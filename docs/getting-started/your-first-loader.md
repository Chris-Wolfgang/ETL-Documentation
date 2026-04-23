# Your First Loader

This guide walks through building a complete loader from scratch, using `JsonLineLoader` from `Wolfgang.Etl.Json` as a real-world example.

A loader is the final step in an ETL pipeline. It receives items as an `IAsyncEnumerable<T>` and writes them to a destination.

**"Destination" is broader than it sounds.** The obvious cases are files, databases, and HTTP APIs, but a loader can write anywhere a consumer sinks data. For example:

- An `SmtpLoader` or `MimeKitLoader` that receives `MailMessage` / `MimeMessage` items from a transformer and sends each one as an email.
- A `ServiceBusLoader` that posts each item to an Azure Service Bus topic.
- A `ConsoleLoader` that writes each item's formatted representation to stdout for diagnostics.
- A `WebhookLoader` that POSTs each item to an external HTTP endpoint.

If your use case feels too unusual to be an ETL, it probably still fits the pipeline model. "Read records somewhere, convert them to the right shape, deliver them somewhere else" is ETL, whether the delivery is a SQL `INSERT` or an SMTP `DATA`.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```


## Interfaces vs. the Base Class

There are two ways to build a loader:

1. **Inherit from `LoaderBase<TDestination, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `LoadWorkerAsync` method. This guide focuses on this path.
2. **Implement one of the loader interfaces directly** — for full control. You get to decide exactly how every piece of the loader behaves, at the cost of implementing everything yourself.

### The Four Loader Interfaces

See [Architecture](../architecture.md#interfaces) for the full diamond structure shared across all three stages. For loaders specifically:

| Interface | What it adds |
|-----------|--------------|
| `ILoadAsync<TDestination>` | `LoadAsync(IAsyncEnumerable<TDestination>)` — the bare minimum |
| `ILoadWithCancellationAsync<TDestination>` | Adds a `CancellationToken` overload |
| `ILoadWithProgressAsync<TDestination, TProgress>` | Adds an `IProgress<TProgress>` overload |
| `ILoadWithProgressAndCancellationAsync<TDestination, TProgress>` | Adds an overload with both |

Implement only the interface your loader actually needs — a loader that does not meaningfully support cancellation should not implement a cancellation interface.

### When to Use the Base Class

Use `LoaderBase<TDestination, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, honors `SkipItemCount` and `MaximumItemCount` consistently, and gives you the timer-injection pattern needed to test progress reporting.

Hand-implementing the interfaces is valid and supported — for example, if you are wrapping a third-party writer with its own async loop and cannot match the base-class lifecycle — but you are responsible for matching the contract that other loaders implement. The contract test base classes in `TestKit.Xunit` (see Step 6) verify behavior consistent with the base class, so deliberate deviations may fail those tests by design.


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
    private int _progressTimerWired;
```

The guard is an `int` rather than a `bool` so it can be set atomically with `Interlocked.CompareExchange` in Step 5 — the `CreateProgressTimer` override may be entered from more than one thread if a caller starts multiple progress-reporting operations concurrently.


## Why Streams, Not Files

Notice that the loader's field is `Stream _stream`, not `string _filePath`. Whenever possible favor using streams rather than file names for the following reasons:

1. **Testable without the filesystem.** A test can hand a loader an empty `MemoryStream` and assert on its contents afterwards. No temp directories, no cleanup, no flaky tests from leftover files.

2. **Composable with whatever the caller has.** The caller chooses where the bytes go — the loader never needs to know:

    - A file: `File.Create("output.jsonl")`
    - An in-memory buffer: `new MemoryStream()`
    - A web response stream handed to the loader by an ASP.NET endpoint
    - A network stream to a remote service

3. **Working with compressed files and streams becomes very simple.** Wrap the underlying stream in a `GZipStream`, `BrotliStream`, or any other wrapper stream and hand it to the loader — no changes to the loader itself:

    ```csharp
    using var fileStream = File.Create("output.jsonl.gz");
    using var gzip = new GZipStream(fileStream, CompressionLevel.Optimal);

    var loader = new JsonLineLoader<Contact>(gzip, logger);

    await loader.LoadAsync
    (
        transformer.TransformAsync(extractor.ExtractAsync(ct), ct),
        ct
    );
    ```

    The loader sees a plain `Stream` and writes byte-by-byte as normal. Compression is entirely a caller-side concern.

Accepting a `Stream` rather than a path costs nothing at the call site (`File.Create(path)` is one line) but everything described above is lost the moment a loader takes a `string filePath`.


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
