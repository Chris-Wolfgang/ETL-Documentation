# Progress Reporting

Every ETL component built on the Abstractions base classes can report progress to a caller-supplied `IProgress<TProgress>`. This guide covers how to consume progress, tune how often it fires, define custom report types, and integrate with a console UI. For the TestKit patterns used to assert on progress callbacks in unit tests, see [TestKit](../reference/testkit.md).


## The IProgress pattern

All ETL components support the standard .NET `IProgress<T>` pattern. Each component provides four overloads of its main method:

| Overload | Description |
|----------|-------------|
| `ExtractAsync()` | No cancellation, no progress |
| `ExtractAsync(CancellationToken)` | Cancellation only |
| `ExtractAsync(IProgress<TProgress>)` | Progress only |
| `ExtractAsync(IProgress<TProgress>, CancellationToken)` | Both |

The same four overloads exist for `LoadAsync` and `TransformAsync`.


## Basic progress reporting

Pass an `IProgress<TProgress>` to receive periodic progress callbacks:

```csharp
var progress = new Progress<FixedWidthReport>(report =>
{
    Console.WriteLine
    (
        $"Extracted {report.CurrentItemCount} items, " +
        $"line {report.CurrentLineNumber}"
    );
});

await foreach (var record in extractor.ExtractAsync(progress, cancellationToken))
{
    Process(record);
}
```

For loaders:

```csharp
var loadProgress = new Progress<FixedWidthReport>(report =>
{
    Console.WriteLine($"Loaded {report.CurrentItemCount} records.");
});

await loader.LoadAsync
(
    extractor.ExtractAsync(cancellationToken),
    loadProgress,
    cancellationToken
);
```

Progress callbacks fire on a background thread at the configured `ReportingInterval`. The component's state properties (`CurrentItemCount`, `CurrentLineNumber`, and so on) are read atomically, so the callback always sees a consistent snapshot.


## ReportingInterval

`ReportingInterval` controls how frequently progress callbacks fire. It is an `int` property on every extractor, transformer, and loader:

```csharp
extractor.ReportingInterval = 500; // Milliseconds
```

The default is 1000 (one second). Lower values give smoother UI updates at the cost of more callback overhead. Higher values reduce overhead for batch workloads where an occasional progress log is enough.


## Report types

Every progress type derives from the `Report` base class in `Wolfgang.Etl.Abstractions`:

```csharp
public record Report
{
    public int CurrentItemCount { get; }
}
```

Each library defines its own `Report` subclass with properties relevant to that format — line numbers for FixedWidth, command text and elapsed time for DbClient, skipped-item counts for JSON and XML, and so on. See the catalog entry for each library for the specific properties:

- [Wolfgang.Etl.FixedWidth](../libraries/fixedwidth.md)
- [Wolfgang.Etl.DbClient](../libraries/dbclient.md)
- [Wolfgang.Etl.Json](../libraries/json.md)
- [Wolfgang.Etl.Xml](../libraries/xml.md)


## Monitoring all pipeline stages

Each stage reports its own progress independently. Pass a separate `IProgress<TProgress>` to each component to monitor the whole pipeline:

```csharp
var extractProgress = new Progress<FixedWidthReport>(r =>
    logger.LogInformation("Extracted: {Count} items", r.CurrentItemCount)
);

var transformProgress = new Progress<Report>(r =>
    logger.LogInformation("Transformed: {Count} items", r.CurrentItemCount)
);

var loadProgress = new Progress<JsonReport>(r =>
    logger.LogInformation("Loaded: {Count} items", r.CurrentItemCount)
);

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(extractProgress, cancellationToken),
        transformProgress,
        cancellationToken
    ),
    loadProgress,
    cancellationToken
);
```

Each stage's progress type can differ — the extractor reports `FixedWidthReport`, the transformer reports `Report`, and the loader reports `JsonReport`. The framework lets each component use its own report type.


## Custom progress types

To add domain-specific information to progress reports, create a custom report type and override `CreateProgressReport`:

```csharp
public record ImportProgress : Report
{
    public ImportProgress
    (
        int currentItemCount,
        int skippedCount,
        long lineNumber,
        string currentFileName
    )
        : base(currentItemCount)
    {
        SkippedCount = skippedCount;
        LineNumber = lineNumber;
        CurrentFileName = currentFileName;
    }



    public int SkippedCount { get; }
    public long LineNumber { get; }
    public string CurrentFileName { get; }
}

public class MyExtractor : FixedWidthExtractor<CustomerRecord, ImportProgress>
{
    public string CurrentFileName { get; set; } = string.Empty;



    public MyExtractor(TextReader reader) : base(reader) { }



    protected override ImportProgress CreateProgressReport() =>
        new
        (
            CurrentItemCount,
            CurrentSkippedItemCount,
            CurrentLineNumber,
            CurrentFileName
        );
}
```

The same pattern works for transformers and loaders — subclass the base class you need and override `CreateProgressReport` to return your custom report type.


## Progress bar integration

A worked example integrating progress reporting with a console progress bar:

```csharp
var totalLines = File.ReadLines("data.txt").Count();

var progress = new Progress<FixedWidthReport>(report =>
{
    var percent = totalLines > 0
        ? (double)report.CurrentLineNumber / totalLines * 100
        : 0;

    var bar = new string('#', (int)(percent / 2));
    Console.Write
    (
        $"\r[{bar,-50}] {percent:F1}% " +
        $"({report.CurrentItemCount} items, " +
        $"{report.CurrentSkippedItemCount} skipped)"
    );
});

extractor.ReportingInterval = 200; // Update 5 times per second

await foreach (var record in extractor.ExtractAsync(progress, cancellationToken))
{
    await ProcessAsync(record);
}

Console.WriteLine(); // New line after progress bar
Console.WriteLine($"Done. {extractor.CurrentItemCount} items extracted.");
```

Libraries like [Spectre.Console](https://spectreconsole.net/) provide richer progress widgets that plug in the same way — route each progress callback into your chosen UI library.


## Testing progress reporting

Progress callbacks fire asynchronously, which makes them awkward to assert on directly. The TestKit package provides:

- **`ManualProgressTimer`** — lets your test fire the progress timer on demand, making callback timing deterministic.
- **Contract test base classes** (`ExtractorBaseContractTests`, `LoaderBaseContractTests`, `TransformerBaseContractTests`) — automatically verify that your component's progress callbacks fire correctly without you writing the assertions yourself.

See [TestKit](../reference/testkit.md) for the full timer-injection pattern. The component-side setup (fields, internal constructor, `CreateProgressTimer` override) is shown in [Your First Extractor](../getting-started/your-first-extractor.md#step-6-override-createprogresstimer) and [Your First Loader](../getting-started/your-first-loader.md#step-5-createprogressreport-and-createprogresstimer).


## See Also

- [Pipeline Composition](pipeline-composition.md) — the full pipeline-level progress example
- [Performance](performance.md) — tuning `ReportingInterval` for throughput
- [TestKit](../reference/testkit.md) — asserting on progress callbacks in tests
