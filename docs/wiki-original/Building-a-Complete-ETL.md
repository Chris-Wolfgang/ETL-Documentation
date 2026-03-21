# Building a Complete ETL

This guide shows how to wire an extractor, transformer, and loader together into a complete ETL pipeline. The design uses lazy evaluation and a pull-based model -- the loader drives the pipeline, pulling items through the transformer from the extractor.

## The Pull Model

```
Loader.LoadAsync(transformer.TransformAsync(extractor.ExtractAsync()))
```

When the loader starts enumerating items:
1. It pulls from the transformer's `IAsyncEnumerable<TDestination>`
2. The transformer pulls from the extractor's `IAsyncEnumerable<TSource>`
3. The extractor reads from the data source and yields items one at a time

No buffering. No intermediate collections. Items flow through the pipeline one at a time, keeping memory usage constant regardless of dataset size.


## Defining Your Data Types

An ETL typically needs:
- A **source type** representing the raw data from the extractor
- A **destination type** representing the data the loader writes
- Optionally, they can be the same type if no transformation is needed

```csharp
// Source: what the extractor yields
public record SourcePerson
{
    public string? first_name { get; set; }
    public string? last_name { get; set; }
    public string? date_of_birth { get; set; }
}

// Destination: what the loader writes
public record DestinationContact
{
    public string? FullName { get; set; }
    public DateTime? BirthDate { get; set; }
}
```


## Building the Transformer

The transformer converts `TSource` to `TDestination`. It inherits from `TransformerBase<TSource, TDestination, TProgress>`:

```csharp
public class PersonToContactTransformer
    : TransformerBase<SourcePerson, DestinationContact, Report>
{
    protected override async IAsyncEnumerable<DestinationContact> TransformWorkerAsync
    (
        IAsyncEnumerable<SourcePerson> items,
        [EnumeratorCancellation] CancellationToken token
    )
    {
        await foreach (var person in items.WithCancellation(token).ConfigureAwait(false))
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

            var contact = new DestinationContact
            {
                FullName = $"{person.first_name} {person.last_name}",
                BirthDate = DateTime.TryParse(person.date_of_birth, out var dob) ? dob : null,
            };

            IncrementCurrentItemCount();
            yield return contact;
        }
    }

    protected override Report CreateProgressReport() => new(CurrentItemCount);
}
```

The same rules apply as extractors:
- Call `IncrementCurrentItemCount()` exactly once per yielded item
- Implement `SkipItemCount` and `MaximumItemCount` yourself
- Respect the `CancellationToken`
- Use `[EnumeratorCancellation]` on the token parameter


## Wiring the Pipeline

### Basic pipeline (no progress, no cancellation)

```csharp
// Extract from JSONL
using var sourceStream = File.OpenRead("people.jsonl");
var extractor = new JsonLineExtractor<SourcePerson>(sourceStream, extractorLogger);

// Transform
var transformer = new PersonToContactTransformer();

// Load to JSONL
using var destStream = File.Create("contacts.jsonl");
var loader = new JsonLineLoader<DestinationContact>(destStream, loaderLogger);

// Wire and run
await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync()
    )
);
```

### With cancellation

```csharp
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromMinutes(5));

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cts.Token),
        cts.Token
    ),
    cts.Token
);
```

### With progress reporting

```csharp
var extractProgress = new Progress<JsonReport>(r =>
    Console.WriteLine($"Extracted: {r.CurrentItemCount}")
);

var transformProgress = new Progress<Report>(r =>
    Console.WriteLine($"Transformed: {r.CurrentItemCount}")
);

var loadProgress = new Progress<JsonReport>(r =>
    Console.WriteLine($"Loaded: {r.CurrentItemCount}")
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

### With skip and maximum item counts

```csharp
// Skip the first 1000 items and extract only the next 500
extractor.SkipItemCount = 1000;
extractor.MaximumItemCount = 500;

// You can also set these on the transformer or loader independently
transformer.ReportingInterval = 5000;  // Report every 5 seconds
```


## Skipping the Transformer

If you do not need to transform data -- for example, reading JSON and writing it to a different JSON format -- you can pass the extractor output directly to the loader:

```csharp
using var sourceStream = File.OpenRead("data.jsonl");
var extractor = new JsonLineExtractor<Person>(sourceStream, extractorLogger);

using var destStream = File.Create("output.json");
var loader = new JsonSingleStreamLoader<Person>(destStream, loaderLogger);

// No transformer needed
await loader.LoadAsync
(
    extractor.ExtractAsync(cancellationToken),
    cancellationToken
);
```

This works because the extractor's `IAsyncEnumerable<Person>` satisfies the loader's `IAsyncEnumerable<Person>` parameter directly.


## Chaining Transformers

You can chain multiple transformers when the transformation is complex. Each transformer's output feeds into the next:

```csharp
var step1 = new CleanDataTransformer();    // SourcePerson -> CleanPerson
var step2 = new EnrichTransformer();       // CleanPerson  -> EnrichedPerson
var step3 = new MapToContactTransformer(); // EnrichedPerson -> DestinationContact

await loader.LoadAsync
(
    step3.TransformAsync
    (
        step2.TransformAsync
        (
            step1.TransformAsync
            (
                extractor.ExtractAsync(cancellationToken),
                cancellationToken
            ),
            cancellationToken
        ),
        cancellationToken
    ),
    cancellationToken
);
```

Each transformer is independently testable, reusable, and composable.


## Interchangeability

Because all extractors implement the same interfaces, you can swap data sources without changing the rest of the pipeline:

```csharp
// Read from JSONL
var extractor = new JsonLineExtractor<SourcePerson>(jsonStream, logger);

// Or read from a JSON array -- same TRecord, different source format
var extractor = new JsonSingleStreamExtractor<SourcePerson>(jsonStream, logger);

// Or read from multiple JSON files -- same TRecord, different I/O model
var extractor = new JsonMultiStreamExtractor<SourcePerson>(fileStreams, logger);
```

The transformer and loader do not care where the data came from. Similarly, you can swap the loader:

```csharp
// Write to JSONL
var loader = new JsonLineLoader<DestinationContact>(stream, logger);

// Or write to a JSON array
var loader = new JsonSingleStreamLoader<DestinationContact>(stream, logger);

// Or write each item to its own file
var loader = new JsonMultiStreamLoader<DestinationContact>
(
    contact => File.Create($"output/{contact.FullName}.json"),
    logger
);
```


## Error Handling

Each component is responsible for handling errors within its domain:
- The **extractor** handles source read failures (file not found, network errors, malformed data)
- The **transformer** handles conversion failures (invalid values, missing fields)
- The **loader** handles write failures (disk full, permissions, API errors)

Unhandled exceptions propagate up through the async enumeration chain and surface at the `LoadAsync` call site.


## Testing the Pipeline

Test each component in isolation using the contract test base classes from [TestKit](TestKit), then write a small integration test that wires them together:

```csharp
[Fact]
public async Task Pipeline_extracts_transforms_and_loads()
{
    // Arrange
    var sourceJson = "[{\"first_name\":\"Alice\",\"last_name\":\"Smith\"}]";
    var sourceStream = new MemoryStream(Encoding.UTF8.GetBytes(sourceJson));
    var destStream = new MemoryStream();

    var extractor = new JsonSingleStreamExtractor<SourcePerson>
    (
        sourceStream,
        NullLogger<JsonSingleStreamExtractor<SourcePerson>>.Instance
    );

    var transformer = new PersonToContactTransformer();

    var loader = new JsonLineLoader<DestinationContact>
    (
        destStream,
        NullLogger<JsonLineLoader<DestinationContact>>.Instance
    );

    // Act
    await loader.LoadAsync
    (
        transformer.TransformAsync
        (
            extractor.ExtractAsync()
        )
    );

    // Assert
    destStream.Position = 0;
    using var reader = new StreamReader(destStream);
    var output = await reader.ReadToEndAsync();

    Assert.Contains("Alice Smith", output);
}
```


## See Also

- [Extractor](Extractor) -- API reference
- [Transformer](Transformer) -- API reference
- [Loader](Loader) -- API reference
- [TestKit](TestKit) -- test doubles and contract tests
- [Building an Extractor](Building-an-Extractor) -- step-by-step extractor guide
- [Building a Loader](Building-a-Loader) -- step-by-step loader guide
