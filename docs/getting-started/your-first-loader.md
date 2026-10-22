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
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.26.1" />
```


## Interfaces vs. the Base Class

There are two ways to build a loader:

1. **Inherit from `LoaderBase<TDestination, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, configuration validation, progress timer setup) and leaves you to implement a single `LoadWorkerAsync` method. This guide focuses on this path.
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

Use `LoaderBase<TDestination, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, validates `SkipItemCount`, `MaximumItemCount` and `ReportingInterval`, runs the progress timer, and lets the TestKit contract tests drive progress deterministically. If you do not need a custom progress type, inherit `LoaderBase<TDestination>` instead (progress type fixed to `Report`, `CreateProgressReport()` supplied).

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
    private readonly JsonSerializerOptions? _serializerOptions;
    private readonly ILogger _logger;
```

As with extractors, the stage-level configuration (`SkipItemCount`, `MaximumItemCount`, `ReportingInterval`, error policy) arrives in a `LoaderOptions` record passed to the base constructor, not in fields of your own.


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

The same pattern as extractors — a minimal constructor and a full one that passes the `LoaderOptions` to the base class:

```csharp
    public JsonLineLoader
    (
        Stream stream,
        ILogger<JsonLineLoader<TRecord>>? logger = null
    )
        : this(stream, serializerOptions: null, options: null, logger)
    {
    }



    public JsonLineLoader
    (
        Stream stream,
        JsonSerializerOptions? serializerOptions,
        LoaderOptions? options = null,
        ILogger<JsonLineLoader<TRecord>>? logger = null
    )
        : base(options)
    {
        _stream = stream ?? throw new ArgumentNullException(nameof(stream));
        _serializerOptions = serializerOptions;
        _logger = (ILogger?)logger ?? NullLogger.Instance;
    }
```

The base constructor validates the `LoaderOptions` values and throws `ArgumentOutOfRangeException` for out-of-range ones; `null` means "use the defaults".


## Step 4: Implement LoadWorkerAsync

Unlike `ExtractWorkerAsync` which returns `IAsyncEnumerable<T>`, `LoadWorkerAsync` returns `Task`. It receives items and writes them to the destination:

```csharp
    protected override async Task LoadWorkerAsync
    (
        IAsyncEnumerable<TRecord> items,
        CancellationToken token
    )
    {
        // Do not pull even one item from the source when the run is already cancelled
        token.ThrowIfCancellationRequested();

        // Explicit encoding and buffer size: the (Stream, leaveOpen:) shorthand is .NET Core 3.0+ only
        using var writer = new StreamWriter(_stream, new UTF8Encoding(false), 4096, leaveOpen: true);

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

            var json = JsonSerializer.Serialize(item, _serializerOptions);
            await writer.WriteLineAsync(json).ConfigureAwait(false);

            IncrementCurrentItemCount();
        }

        await writer.FlushAsync().ConfigureAwait(false);
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
2. **Implement `SkipItemCount` and `MaximumItemCount` yourself** (the base class validates them but does not apply them)
3. **Respect the `CancellationToken`** — check it before the first item as well as inside the loop, so an already-cancelled run reads nothing from the source
4. **Use `.ConfigureAwait(false)` on all await calls**


## Step 5: CreateProgressReport

Identical pattern to extractors:

```csharp
    protected override JsonReport CreateProgressReport() =>
        new
        (
            CurrentItemCount,
            CurrentSkippedItemCount
        );
}
```

No timer plumbing is needed: the base class runs the production timer, and TestKit drives progress in tests through the base class (see [TestKit § Progress Timing in Tests](../reference/testkit.md#progress-timing-in-tests)). Earlier versions of this guide added an `IProgressTimer` field, an internal constructor and a `CreateProgressTimer` override for that; leave them out of new loaders.


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



    protected override JsonLineLoader<PersonRecord> CreateSut
    (
        int itemCount,
        int maximumItemCount,
        int skipItemCount,
        int reportingInterval
    ) =>
        new
        (
            new MemoryStream(),
            serializerOptions: null,
            new LoaderOptions
            {
                MaximumItemCount = maximumItemCount,
                SkipItemCount = skipItemCount,
                ReportingInterval = reportingInterval,
            }
        );



    protected override IReadOnlyList<PersonRecord> CreateSourceItems() => SourceItems;
}
```

!!! note
    `CreateSut` for loaders creates a fresh `MemoryStream` each time. The `itemCount` parameter tells you how many items will be loaded — useful for loaders that pre-allocate. For a JSONL loader, you can ignore it. Pass the three configuration values straight through: some contract tests use out-of-range values on purpose and expect construction to throw.

Those two factory methods give you the 46 tests of the `LoaderBase` contract.


## Multi-TFM Considerations

The `LoadWorkerAsync` above uses the four-argument `StreamWriter` constructor, `WriteLineAsync(string)` and `FlushAsync()`, which exist on every target framework, and observes cancellation through `token.ThrowIfCancellationRequested()` on each item. The token-accepting overloads are newer: `TextWriter.WriteLineAsync(ReadOnlyMemory<char>, CancellationToken)` needs netstandard2.1 / .NET Core 3.0+, and `StreamWriter.FlushAsync(CancellationToken)` needs .NET 8+. To use them where available, guard each call with the matching `_OR_GREATER` symbol:

```csharp
#if NET8_0_OR_GREATER
        await writer.FlushAsync(token).ConfigureAwait(false);
#else
        await writer.FlushAsync().ConfigureAwait(false);
#endif
```


## Checklist

- [ ] Inherit `LoaderBase<TRecord, TProgress>` (or `LoaderBase<TRecord>`) with `where TRecord : notnull`
- [ ] Accept a `LoaderOptions?` (or a record derived from it) and pass it to `base(options)`
- [ ] Implement `LoadWorkerAsync(IAsyncEnumerable<TRecord>, CancellationToken)`
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per loaded item
- [ ] Implement `SkipItemCount` logic
- [ ] Implement `MaximumItemCount` logic
- [ ] Respect `CancellationToken`
- [ ] Test class inherits `LoaderBaseContractTests<TSut, TItem, TProgress>` and implements the four-argument `CreateSut`
- [ ] `CreateSourceItems()` returns at least 5 items
- [ ] Add domain-specific tests for output format and edge cases


## See Also

- [Your First Extractor](your-first-extractor.md) — the companion guide for extractors
- [Your First Transformer](your-first-transformer.md) — the companion guide for transformers
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractor, transformer, and loader together
- [Architecture](../architecture.md) — base classes and design patterns
