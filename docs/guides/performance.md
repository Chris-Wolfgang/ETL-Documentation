# Performance

This guide covers framework-level performance characteristics and practical tips that apply across the libraries. For per-library optimizations (span-based parsing, compiled delegates, zero-copy field slicing, and so on), see each library's README.


## Memory Model

The framework's pull-based `IAsyncEnumerable<T>` pipeline processes items one at a time. No intermediate collections are created between pipeline stages. This means:

- **Constant peak live memory** regardless of file size
- Gen2 heap pressure stays flat from 0 to 1M+ records
- The only scaling factor is the size of a single record, not the total dataset

This holds true as long as you do not materialize the `IAsyncEnumerable` into a list or array. The pipeline is designed for streaming — items enter, flow through, and exit without accumulating.


## General Performance Tips

### Use the Stream constructor when available

When a library offers both a `Stream` constructor and a `TextReader` constructor, prefer the `Stream` one — libraries typically configure a larger read buffer internally than the `StreamReader` default (often 1 KB). If you need to use a `TextReader` constructor, wrap your stream in a `StreamReader` with an explicitly larger buffer:

```csharp
// Preferred: use the Stream constructor
using var stream = File.OpenRead("data.txt");
var extractor = new FixedWidthExtractor<MyRecord, FixedWidthReport>(stream, logger);

// Alternative: manual large buffer with TextReader constructor
using var stream = File.OpenRead("data.txt");
using var reader = new StreamReader
(
    stream,
    encoding: Encoding.UTF8,
    detectEncodingFromByteOrderMarks: true,
    bufferSize: 65536
);
var extractor = new FixedWidthExtractor<MyRecord, FixedWidthReport>(reader, logger);
```

### Use SkipItemCount and MaximumItemCount instead of LINQ

The base class properties `SkipItemCount` and `MaximumItemCount` are implemented inside `ExtractWorkerAsync` / `LoadWorkerAsync`, where they can short-circuit the read loop and avoid reading or processing items that will be discarded.

LINQ's `Skip()` and `Take()` operators, by contrast, still enumerate through the skipped items — they just discard them after extraction:

```csharp
// Good: skips at the source level — lines are read but not parsed
extractor.SkipItemCount = 1000;
extractor.MaximumItemCount = 500;

// Bad: parses all 1000 skipped records before discarding them
var items = extractor.ExtractAsync(cancellationToken)
    .Skip(1000)
    .Take(500);
```

### Avoid materializing IAsyncEnumerable into lists

The entire framework is designed for streaming. Materializing the async enumerable into a `List<T>` or array defeats the constant-memory model:

```csharp
// Bad: loads entire dataset into memory
var allRecords = new List<MyRecord>();
await foreach (var record in extractor.ExtractAsync(cancellationToken))
{
    allRecords.Add(record);
}
await loader.LoadAsync(allRecords.ToAsyncEnumerable(), cancellationToken);

// Good: stream directly
await loader.LoadAsync
(
    extractor.ExtractAsync(cancellationToken),
    cancellationToken
);
```

### Keep record types simple

Record types with complex computed properties, lazy initialization, or heavy constructor logic add overhead per item. Use simple POCOs with auto-properties:

```csharp
// Good: simple POCO
public record CustomerRecord
{
    [FixedWidthField(0, 30)]
    public string? Name { get; set; }

    [FixedWidthField(1, 10)]
    public int Id { get; set; }
}

// Bad: computed property runs on every access
public record CustomerRecord
{
    [FixedWidthField(0, 30)]
    public string? Name { get; set; }

    [FixedWidthField(1, 10)]
    public int Id { get; set; }

    // This getter runs expensive logic
    public string DisplayName => ComputeDisplayName(Name, Id);
}
```

### Tune ReportingInterval

Progress reporting fires on a background thread at `ReportingInterval` milliseconds. A very low interval (e.g., 50 ms) can add contention on shared state reads. For maximum throughput in batch scenarios, increase the interval or omit progress reporting entirely:

```csharp
// High-throughput batch: no progress reporting
await loader.LoadAsync
(
    extractor.ExtractAsync(cancellationToken),
    cancellationToken
);

// Interactive: report every 500 ms
extractor.ReportingInterval = 500;
await foreach (var record in extractor.ExtractAsync(progress, cancellationToken))
{
    Process(record);
}
```


## Profiling Your Pipeline

When optimizing, measure before and after. The framework's constant-memory model means you should focus on:

1. **Throughput** (records/second) — use `Stopwatch` around the `LoadAsync` call
2. **Allocation rate** — use `dotnet-counters` or BenchmarkDotNet with `MemoryDiagnoser`
3. **GC pressure** — monitor Gen0/Gen1/Gen2 collection counts

```csharp
var sw = Stopwatch.StartNew();

await loader.LoadAsync
(
    extractor.ExtractAsync(cancellationToken),
    cancellationToken
);

sw.Stop();

logger.LogInformation
(
    "Processed {Count} records in {Elapsed:F2}s ({Rate:F0} records/sec)",
    extractor.CurrentItemCount,
    sw.Elapsed.TotalSeconds,
    extractor.CurrentItemCount / sw.Elapsed.TotalSeconds
);
```


## Library-specific performance

Each library has its own performance characteristics and optimizations — span-based parsing, compiled accessor delegates, larger I/O buffers, and multi-TFM fast paths on net8.0+ where span APIs are available. The specifics live in each library's README, along with benchmark numbers for that library's workloads:

- [ETL-FixedWidth README](https://github.com/Chris-Wolfgang/ETL-FixedWidth#readme)
- [ETL-Json README](https://github.com/Chris-Wolfgang/ETL-Json#readme)
- [ETL-Xml README](https://github.com/Chris-Wolfgang/ETL-Xml#readme)
- [ETL-DbClient README](https://github.com/Chris-Wolfgang/ETL-DbClient#readme)

If you are processing high volumes and performance is critical, target `net8.0` or later — most libraries have span-based fast paths on newer TFMs that fall back to string-based implementations on netstandard2.0.


## See Also

- [Pipeline Composition](pipeline-composition.md) — streaming pipeline architecture
- [Custom Parsers & Converters](custom-parsers.md) — customizing parsing and serialization hooks
- [Progress Reporting](progress-reporting.md) — tuning `ReportingInterval`
