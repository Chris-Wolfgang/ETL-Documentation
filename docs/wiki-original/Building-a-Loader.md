# Building a Loader

This guide walks through building a complete loader from scratch, using `JsonLineLoader` from the `Wolfgang.Etl.Json` library as a real-world example. By the end, you will have a fully tested loader that writes JSONL data to a stream.

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


## Why Streams, Not Files

Notice that the loader's field is `Stream _stream`, not `string _filePath`. Whenever possible favor using streams rather than file names for the following reasons:

1. **Testable without the filesystem.** A test can build a `MemoryStream` with exactly the data it wants to feed to an extractor, or hand a loader an empty `MemoryStream` and assert on its contents afterwards. No temp directories, no cleanup, no flaky tests from leftover files.

2. **Composable with whatever the caller has.** The caller chooses where the bytes come from or go to -- the loader never needs to know:

   - A file: `File.OpenRead("data.jsonl")`
   - An in-memory buffer: `new MemoryStream(bytes)`
   - A web request body: `await HttpResponseMessage.Content.ReadAsStreamAsync()`
   - A web *response* to a caller-supplied `Stream`: the caller hands you `Response.Body`

3. **Working with compressed files and streams becomes very simple.** Wrap the underlying stream in a `GZipStream`, `BrotliStream`, or any other wrapper stream and hand it to the loader -- no changes to the loader itself:

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

   See [Reading and Writing Compressed Files and Streams](Reading-and-Writing-Compressed-Files-and-Streams) for the full set of examples including read-side decompression, Brotli, Deflate, compression-level tradeoffs, and the dispose-ordering pitfall.

Accepting a `Stream` rather than a path costs nothing at the call site (`File.Create(path)` is one line) but everything described above is lost the moment a loader takes a `string filePath`.


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
