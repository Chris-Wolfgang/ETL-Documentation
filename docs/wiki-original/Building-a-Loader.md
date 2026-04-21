# Building a Loader

This guide walks through building a complete loader from scratch, using `JsonLineLoader` from the `Wolfgang.Etl.Json` library as a real-world example. By the end, you will have a fully tested loader that writes JSONL data to a stream.

A loader is the final step in an ETL pipeline. It receives items as an `IAsyncEnumerable<T>` and writes them to a destination -- a file, database, API, message queue, or any other target.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```


## Interfaces vs. the Base Class

There are two ways to build a loader:

1. **Inherit from `LoaderBase<TDestination, TProgress>`** -- the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `LoadWorkerAsync` method. This guide focuses on this path.
2. **Implement one of the loader interfaces directly** -- for full control. You get to decide exactly how every piece of the loader behaves, at the cost of implementing everything yourself.

### The Four Loader Interfaces

The loader interfaces form a diamond -- start with `ILoadAsync<TDestination>` and add only the capabilities you need:

```
            ILoadAsync<TDestination>
                  /        \
                 /          \
ILoadWithCancellation    ILoadWithProgress
     Async<T>             Async<T, TProgress>
                 \          /
                  \        /
       ILoadWithProgressAndCancellation
                Async<T, TProgress>
```

| Interface | What it adds |
|-----------|--------------|
| `ILoadAsync<TDestination>` | `LoadAsync(IAsyncEnumerable<TDestination>)` -- the bare minimum |
| `ILoadWithCancellationAsync<TDestination>` | Adds a `CancellationToken` overload |
| `ILoadWithProgressAsync<TDestination, TProgress>` | Adds an `IProgress<TProgress>` overload |
| `ILoadWithProgressAndCancellationAsync<TDestination, TProgress>` | Adds an overload with both |

The cancellation and progress interfaces both inherit from `ILoadAsync<TDestination>`. The combined interface inherits from both, which is the diamond. Implement only the interface your loader actually needs -- a loader that does not meaningfully support cancellation should not implement a cancellation interface.

### When to Use the Base Class

Use `LoaderBase<TDestination, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, honors `SkipItemCount` and `MaximumItemCount` consistently, and gives you the timer-injection pattern needed to test progress reporting (see [TestKit](TestKit)).

Hand-implementing the interfaces is valid and supported -- for example, if you are wrapping a third-party writer with its own async loop and cannot match the base-class lifecycle -- but you are responsible for matching the contract that other loaders implement. The contract test base classes in [TestKit.Xunit](TestKit) verify behavior consistent with the base class, so deliberate deviations may fail those tests by design.


## Step 1: Define Your Progress Report

Reuse the same progress report as your extractor if they share a library, or create a new one:

```csharp
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


## Step 2: Create the Loader Class

Inherit from `LoaderBase<TRecord, TProgress>`:

```csharp
public class JsonLineLoader<TRecord> : LoaderBase<TRecord, JsonReport>
    where TRecord : notnull
{
    private readonly Stream _stream;
    private readonly JsonSerializerOptions? _options;
    private readonly ILogger _logger;
    private readonly IProgressTimer? _progressTimer;
    private bool _progressTimerWired;
```


## Step 3: Constructors

The same three-constructor pattern as extractors:

```csharp
// Public: minimal
public JsonLineLoader
(
    Stream stream,
    ILogger<JsonLineLoader<TRecord>> logger
)
{
    _stream = stream ?? throw new ArgumentNullException(nameof(stream));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    _options = null;
}

// Public: with options
public JsonLineLoader
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonLineLoader<TRecord>> logger
)
{
    _stream = stream ?? throw new ArgumentNullException(nameof(stream));
    _options = options ?? throw new ArgumentNullException(nameof(options));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
}

// Internal: for timer injection in tests
internal JsonLineLoader
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


## Step 4: Implement LoadWorkerAsync

Unlike `ExtractWorkerAsync` which returns `IAsyncEnumerable<T>`, `LoadWorkerAsync` returns `Task`. It receives items and writes them to the destination.

```csharp
protected override async Task LoadWorkerAsync
(
    IAsyncEnumerable<TRecord> items,
    CancellationToken token
)
{
    using var writer = new StreamWriter(_stream, leaveOpen: true);

    await foreach (var item in items.WithCancellation(token).ConfigureAwait(false))
    {
        token.ThrowIfCancellationRequested();

        // Handle SkipItemCount
        if (CurrentSkippedItemCount < SkipItemCount)
        {
            IncrementCurrentSkippedItemCount();
            continue;
        }

        // Handle MaximumItemCount
        if (CurrentItemCount >= MaximumItemCount)
        {
            break;
        }

        // Serialize and write the item
        var json = JsonSerializer.Serialize(item, _options);
        await writer.WriteLineAsync(json.AsMemory(), token).ConfigureAwait(false);

        // Count the item
        IncrementCurrentItemCount();
    }

    await writer.FlushAsync(token).ConfigureAwait(false);
}
```

### Key Differences from ExtractWorkerAsync

| Extractor | Loader |
|-----------|--------|
| Returns `IAsyncEnumerable<T>` (yields items) | Returns `Task` (consumes items) |
| Uses `yield return` | Writes to destination |
| Receives no items | Receives `IAsyncEnumerable<T> items` parameter |
| `SkipItemCount` uses a local `skipBudget` counter | `SkipItemCount` compares `CurrentSkippedItemCount < SkipItemCount` |

### Critical Rules (same as extractors)

1. **Call `IncrementCurrentItemCount()` exactly once per loaded item**
2. **Implement `SkipItemCount` and `MaximumItemCount` yourself** -- the base class does not handle these
3. **Respect the `CancellationToken`**
4. **Use `.ConfigureAwait(false)` on all await calls**


## Step 5: Implement CreateProgressReport and CreateProgressTimer

Identical pattern to extractors:

```csharp
protected override JsonReport CreateProgressReport() =>
    new
    (
        CurrentItemCount,
        CurrentSkippedItemCount
    );


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


## Step 6: Write Tests

### Inherit the contract test base class

```csharp
public class JsonLineLoaderTests
    : LoaderBaseContractTests
    <
        JsonLineLoader<PersonRecord>,
        PersonRecord,
        JsonReport
    >
{
    private static readonly IReadOnlyList<PersonRecord> SourceItems = new List<PersonRecord>
    {
        new() { FirstName = "Alice", LastName = "Smith", Age = 30 },
        new() { FirstName = "Bob", LastName = "Jones", Age = 25 },
        new() { FirstName = "Carol", LastName = "White", Age = 35 },
        new() { FirstName = "Dave", LastName = "Brown", Age = 40 },
        new() { FirstName = "Eve", LastName = "Davis", Age = 28 },
    };


    protected override JsonLineLoader<PersonRecord> CreateSut(int itemCount)
    {
        var stream = new MemoryStream();
        return new JsonLineLoader<PersonRecord>
        (
            stream,
            NullLogger<JsonLineLoader<PersonRecord>>.Instance
        );
    }


    protected override IReadOnlyList<PersonRecord> CreateSourceItems() => SourceItems;


    protected override JsonLineLoader<PersonRecord> CreateSutWithTimer
    (
        IProgressTimer timer
    )
    {
        var stream = new MemoryStream();
        return new JsonLineLoader<PersonRecord>
        (
            stream,
            new JsonSerializerOptions(),
            NullLogger<JsonLineLoader<PersonRecord>>.Instance,
            timer
        );
    }
}
```

Note that `CreateSut` for loaders creates a fresh `MemoryStream` each time. The `itemCount` parameter tells you how many items will be loaded, which matters for loaders that pre-allocate (e.g., a fixed-width file loader that sizes a buffer). For a JSONL loader writing to a stream, you can ignore it.

### Add domain-specific tests

Test the output format, round-tripping, and edge cases:

```csharp
[Fact]
public async Task LoadAsync_writes_one_json_object_per_line()
{
    var stream = new MemoryStream();
    var sut = new JsonLineLoader<PersonRecord>
    (
        stream,
        NullLogger<JsonLineLoader<PersonRecord>>.Instance
    );

    var items = new List<PersonRecord>
    {
        new() { FirstName = "Alice", LastName = "Smith", Age = 30 },
        new() { FirstName = "Bob", LastName = "Jones", Age = 25 },
    };

    await sut.LoadAsync(items.ToAsyncEnumerable());

    stream.Position = 0;
    using var reader = new StreamReader(stream);
    var content = await reader.ReadToEndAsync();
    var lines = content.Split(['\n'], StringSplitOptions.RemoveEmptyEntries);

    Assert.Equal(2, lines.Length);

    var person1 = JsonSerializer.Deserialize<PersonRecord>(lines[0]);
    Assert.NotNull(person1);
    Assert.Equal("Alice", person1.FirstName);
}


[Fact]
public async Task LoadAsync_when_empty_sequence_writes_no_json_lines()
{
    var stream = new MemoryStream();
    var sut = new JsonLineLoader<PersonRecord>
    (
        stream,
        NullLogger<JsonLineLoader<PersonRecord>>.Instance
    );

    await sut.LoadAsync(AsyncEnumerable.Empty<PersonRecord>());

    stream.Position = 0;
    using var reader = new StreamReader(stream);
    var content = (await reader.ReadToEndAsync()).Trim();

    Assert.Equal(string.Empty, content);
}
```


## Using Your Loader

```csharp
// Simple usage
using var stream = File.Create("output.jsonl");
var loader = new JsonLineLoader<Person>(stream, logger);

await loader.LoadAsync(items, cancellationToken);

// With skip and maximum
var loader = new JsonLineLoader<Person>(stream, logger);
loader.SkipItemCount = 100;      // Skip first 100 items
loader.MaximumItemCount = 50;    // Then load 50 items

// With progress reporting
var progress = new Progress<JsonReport>(report =>
    Console.WriteLine($"Loaded {report.CurrentItemCount} items")
);

await loader.LoadAsync(items, progress, cancellationToken);
```


## Loader Variants

The `Wolfgang.Etl.Json` library includes three different loader patterns that demonstrate how the same base class supports different destination models:

### Single Stream Loader
Writes all items to one stream as a JSON array (`[{...},{...}]`):

```csharp
using var stream = File.Create("output.json");
var loader = new JsonSingleStreamLoader<Person>(stream, logger);
await loader.LoadAsync(items, cancellationToken);
```

### Multi-Stream Loader
Writes each item to its own stream using a factory function:

```csharp
var loader = new JsonMultiStreamLoader<Person>
(
    person => File.Create($"output/{person.Id}.json"),
    logger
);
await loader.LoadAsync(items, cancellationToken);
```

### Line Loader (JSONL)
Writes one JSON object per line:

```csharp
using var stream = File.Create("output.jsonl");
var loader = new JsonLineLoader<Person>(stream, logger);
await loader.LoadAsync(items, cancellationToken);
```

These demonstrate that the `LoaderBase` pattern is flexible enough to handle single-destination, multi-destination, and streaming output models.


## Checklist

- [ ] Inherit `LoaderBase<TRecord, TProgress>` with `where TRecord : notnull`
- [ ] Implement `LoadWorkerAsync(IAsyncEnumerable<TRecord>, CancellationToken)`
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per loaded item
- [ ] Implement `SkipItemCount` logic (compare `CurrentSkippedItemCount < SkipItemCount`)
- [ ] Implement `MaximumItemCount` logic (check `CurrentItemCount >= MaximumItemCount`)
- [ ] Respect `CancellationToken`
- [ ] Add `_progressTimer` field, `_progressTimerWired` flag, and internal constructor
- [ ] Override `CreateProgressTimer` with duplicate subscription guard
- [ ] Add `InternalsVisibleTo` in `Properties/AssemblyInfo.cs`
- [ ] Test class inherits `LoaderBaseContractTests<TSut, TItem, TProgress>`
- [ ] `CreateSourceItems()` returns at least 5 items
- [ ] Test output format, empty sequences, round-tripping


## See Also

- [Loader](Loader) -- API reference for interfaces and base class
- [TestKit](TestKit) -- contract test base classes and test helpers
- [Building an Extractor](Building-an-Extractor) -- the companion guide for extractors
- [Building a Complete ETL](Building-a-Complete-ETL) -- wiring extractor, transformer, and loader together
