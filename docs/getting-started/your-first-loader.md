# Your First Loader

This guide walks through building a complete loader from scratch, using `JsonLineLoader` from `Wolfgang.Etl.Json` as a real-world example.

A loader is the final step in an ETL pipeline. It receives items as an `IAsyncEnumerable<T>` and writes them to a destination — a file, database, API, message queue, or any other target.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```


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

Unlike `ExtractWorkerAsync` which returns `IAsyncEnumerable<T>`, `LoadWorkerAsync` returns `Task`. It receives items and writes them to the destination:

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

            if (CurrentSkippedItemCount < SkipItemCount)
            {
                IncrementCurrentSkippedItemCount();
                continue;
            }

            if (CurrentItemCount >= MaximumItemCount)
            {
                break;
            }

            var json = JsonSerializer.Serialize(item, _options);
            await writer.WriteLineAsync(json.AsMemory(), token).ConfigureAwait(false);

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

### Critical Rules

1. **Call `IncrementCurrentItemCount()` exactly once per loaded item**
2. **Implement `SkipItemCount` and `MaximumItemCount` yourself**
3. **Respect the `CancellationToken`**
4. **Use `.ConfigureAwait(false)` on all await calls**


## Step 5: CreateProgressReport and CreateProgressTimer

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



    protected override JsonLineLoader<PersonRecord> CreateSut(int itemCount) =>
        new
        (
            new MemoryStream(),
            NullLogger<JsonLineLoader<PersonRecord>>.Instance
        );



    protected override IReadOnlyList<PersonRecord> CreateSourceItems() => SourceItems;



    protected override JsonLineLoader<PersonRecord> CreateSutWithTimer
    (
        IProgressTimer timer
    ) =>
        new
        (
            new MemoryStream(),
            new JsonSerializerOptions(),
            NullLogger<JsonLineLoader<PersonRecord>>.Instance,
            timer
        );
}
```

!!! note
    `CreateSut` for loaders creates a fresh `MemoryStream` each time. The `itemCount` parameter tells you how many items will be loaded — useful for loaders that pre-allocate. For a JSONL loader, you can ignore it.


## Checklist

- [ ] Inherit `LoaderBase<TRecord, TProgress>` with `where TRecord : notnull`
- [ ] Implement `LoadWorkerAsync(IAsyncEnumerable<TRecord>, CancellationToken)`
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per loaded item
- [ ] Implement `SkipItemCount` logic
- [ ] Implement `MaximumItemCount` logic
- [ ] Respect `CancellationToken`
- [ ] Add timer injection pattern (internal constructor, `_progressTimerWired` flag)
- [ ] Override `CreateProgressTimer` with duplicate subscription guard
- [ ] Test class inherits `LoaderBaseContractTests<TSut, TItem, TProgress>`
- [ ] `CreateSourceItems()` returns at least 5 items
- [ ] Add domain-specific tests for output format and edge cases


## See Also

- [Your First Extractor](your-first-extractor.md) — the companion guide for extractors
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractor, transformer, and loader together
- [Architecture](../architecture.md) — base classes and design patterns
