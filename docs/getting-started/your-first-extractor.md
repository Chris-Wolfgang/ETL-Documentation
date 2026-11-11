# Your First Extractor

This guide walks through building a complete extractor from scratch, using `JsonLineExtractor` from `Wolfgang.Etl.Json` as a real-world example.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.26.1" />
```

Plus whatever libraries your extractor needs to read its format (e.g. `System.Text.Json` for JSON) and, for logging, `Microsoft.Extensions.Logging.Abstractions`.


## Interfaces vs. the Base Class

There are two ways to build an extractor:

1. **Inherit from `ExtractorBase<TSource, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, configuration validation, progress timer setup) and leaves you to implement a single `ExtractWorkerAsync` method. This guide focuses on this path.
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

Use `ExtractorBase<TSource, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, validates `SkipItemCount`, `MaximumItemCount` and `ReportingInterval`, runs the progress timer, and lets the TestKit contract tests drive progress deterministically (see Step 6).

If you do not need a custom progress type, inherit `ExtractorBase<TSource>` instead: it fixes the progress type to the built-in `Report` and supplies `CreateProgressReport()` for you.

Hand-implementing the interfaces is valid and supported — for example, if you are wrapping a third-party reader with its own async loop and cannot match the base-class lifecycle — but you are responsible for matching the contract that other extractors implement. The contract test base classes in `TestKit.Xunit` (see Step 6) verify behavior consistent with the base class, so deliberate deviations may fail those tests by design.


## Step 1: Define Your Progress Report

Every extractor needs a progress report type. The base class `Report` from Abstractions provides `CurrentItemCount` (plus timing values). Extend it with format-specific information:

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
    private readonly JsonSerializerOptions? _serializerOptions;
    private readonly ILogger _logger;
```

| Field | Purpose |
|-------|---------|
| `_stream` | The data source your extractor reads from |
| `_serializerOptions` | Format-specific configuration (optional) |
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |

The stage-level configuration (`SkipItemCount`, `MaximumItemCount`, `ReportingInterval`, error policy) is not a field of yours: it arrives in an `ExtractorOptions` record that you pass to the base constructor.


## Step 3: Constructors

A minimal constructor for the common case, plus one that takes everything:

```csharp
    // Public constructor (minimal)
    public JsonLineExtractor
    (
        Stream stream,
        ILogger<JsonLineExtractor<TRecord>>? logger = null
    )
        : this(stream, serializerOptions: null, options: null, logger)
    {
    }



    // Public constructor (full)
    public JsonLineExtractor
    (
        Stream stream,
        JsonSerializerOptions? serializerOptions,
        ExtractorOptions? options = null,
        ILogger<JsonLineExtractor<TRecord>>? logger = null
    )
        : base(options)
    {
        _stream = stream ?? throw new ArgumentNullException(nameof(stream));
        _serializerOptions = serializerOptions;
        _logger = (ILogger?)logger ?? NullLogger.Instance;
    }
```

Passing `options` to `base(options)` is what makes `SkipItemCount`, `MaximumItemCount` and `ReportingInterval` configurable. The base constructor validates them and throws `ArgumentOutOfRangeException` for out-of-range values, and `null` means "use the defaults". A real library usually defines its own options record that derives from `ExtractorOptions` (as `Wolfgang.Etl.Json` does with `JsonLineExtractorOptions`) so format settings and stage settings travel together.


## Step 4: Implement ExtractWorkerAsync

This is the core of your extractor. It is an `async` iterator method that yields items one at a time:

```csharp
    protected override async IAsyncEnumerable<TRecord> ExtractWorkerAsync
    (
        [EnumeratorCancellation] CancellationToken token
    )
    {
        var skipBudget = SkipItemCount;

        using var reader = new StreamReader(_stream, Encoding.UTF8, true, 4096, leaveOpen: true);

        string? line;
        while ((line = await reader.ReadLineAsync().ConfigureAwait(false)) is not null)
        {
            token.ThrowIfCancellationRequested();

            if (string.IsNullOrWhiteSpace(line))
            {
                continue;
            }

            var item = JsonSerializer.Deserialize<TRecord>(line, _serializerOptions);
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

The worker uses `ReadLineAsync()` and the five-argument `StreamReader` constructor because they exist on every target framework; see [Multi-TFM Considerations](#multi-tfm-considerations) for the token-accepting overload on .NET 7+.

### Critical Rules

1. **Call `IncrementCurrentItemCount()` exactly once per yielded item.** Not in helper methods, not conditionally — once, right before or after the `yield return`.

2. **Implement `SkipItemCount` yourself.** The base class validates it but does not skip for you. Use a local `skipBudget` variable.

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
}
```

This method is called by the base class on a timer interval and passed to the `IProgress<T>` callback.

That completes the extractor. There is no timer plumbing to add: the base class creates the production timer, and in tests TestKit attaches a manual timer to the base class directly (see [TestKit § Progress Timing in Tests](../reference/testkit.md#progress-timing-in-tests)).

!!! note "Older guides: CreateProgressTimer and an internal constructor"
    Earlier versions of this guide added an `IProgressTimer` field, an `int _progressTimerWired` guard, an internal constructor taking the timer, a `CreateProgressTimer` override and `InternalsVisibleTo`, so tests could inject a timer. The TestKit contract no longer uses that seam, so new extractors should leave it out.


## Step 6: Write Tests

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
            serializerOptions: null,
            new ExtractorOptions
            {
                MaximumItemCount = maximumItemCount,
                SkipItemCount = skipItemCount,
                ReportingInterval = reportingInterval,
            }
        );



    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;



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

This single class, with just two factory methods, gives you 43 tests that validate your extractor against the full `ExtractorBase` contract. Pass the three configuration values straight through without validating them: some contract tests pass out-of-range values on purpose and expect the constructor to throw.

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

    var sut = new JsonLineExtractor<PersonRecord>(stream);

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

// With skip and maximum: configure them when the extractor is constructed
using var pagedStream = File.OpenRead("data.jsonl");
var paged = new JsonLineExtractor<Person>
(
    pagedStream,
    serializerOptions: null,
    new ExtractorOptions { SkipItemCount = 100, MaximumItemCount = 50 },
    logger
);

// With progress reporting
var progress = new Progress<JsonReport>(report =>
    Console.WriteLine($"Extracted {report.CurrentItemCount} items")
);

await foreach (var person in paged.ExtractAsync(progress, cancellationToken))
{
    // ...
}
```


## Multi-TFM Considerations

Some `CancellationToken` overloads exist only on newer frameworks. `StreamReader.ReadLineAsync(CancellationToken)` arrived in .NET 7, so a library that also targets netstandard2.0, net462 or net5.0 / net6.0 must guard it with the `_OR_GREATER` symbol rather than listing the older targets:

```csharp
#if NET7_0_OR_GREATER
    while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
#else
    while ((line = await reader.ReadLineAsync().ConfigureAwait(false)) is not null)
#endif
```

On the older targets, call `token.ThrowIfCancellationRequested()` inside the loop so cancellation is still observed between lines.


## Checklist

- [ ] Inherit `ExtractorBase<TRecord, TProgress>` (or `ExtractorBase<TRecord>`) with `where TRecord : notnull`
- [ ] Accept an `ExtractorOptions?` (or a record derived from it) and pass it to `base(options)`
- [ ] Implement `ExtractWorkerAsync` with `[EnumeratorCancellation]` on the token
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per yielded item
- [ ] Implement `SkipItemCount` logic in your worker method
- [ ] Implement `MaximumItemCount` logic in your worker method
- [ ] Respect `CancellationToken` (pass to async calls, call `ThrowIfCancellationRequested`)
- [ ] Test class inherits `ExtractorBaseContractTests<TSut, TItem, TProgress>` and implements the four-argument `CreateSut`
- [ ] `CreateExpectedItems()` returns at least 5 items
- [ ] Items use `record` types for value equality
- [ ] Add domain-specific tests for format-specific behavior


## See Also

- [Your First Transformer](your-first-transformer.md) — the companion guide for transformers
- [Your First Loader](your-first-loader.md) — the companion guide for loaders
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractor, transformer, and loader together
- [Architecture](../architecture.md) — base classes and design patterns
