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

The transformer converts `TSource` to `TDestination`. The recommended way to build one is to inherit from `TransformerBase<TSource, TDestination, TProgress>`, which handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `TransformWorkerAsync` method. That is what this guide does.

Inheritance from the base class is **optional**. The same choice exists for extractors and loaders -- see [Building an Extractor](Building-an-Extractor) and [Building a Loader](Building-a-Loader) for the full "Interfaces vs. the Base Class" discussion. You can implement any of the four transformer interfaces (`ITransformAsync`, `ITransformWithCancellationAsync`, `ITransformWithProgressAsync`, `ITransformWithProgressAndCancellationAsync`) directly if you need full control over how the transformer behaves. The four interfaces form the same diamond shape as the extractor and loader interfaces, with the base interface adding only `TransformAsync(IAsyncEnumerable<TSource>)` and each subsequent interface adding cancellation, progress, or both.

The remainder of this guide uses `TransformerBase<TSource, TDestination, TProgress>`:

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


## When You Do Not Need to Transform Data

Every ETL has a transformer stage. When no transformation is needed -- for example, reading a fixed width file and writing it to a CSV file -- do **not** connect the loader directly to the extractor. Skipping the transformer entirely is an antipattern:

- It breaks the three-stage shape the rest of the framework is built around.
- Adding a transformer later (for logging, validation, enrichment, rate limiting, etc.) requires restructuring the pipeline.
- It is harder for readers of the code to see the ETL pattern.

Instead, use `PassThroughTransformer<T>` from `Wolfgang.Etl.Transformers` -- a pass-through transformer that pulls each item from its source enumerable and yields it unchanged:

```csharp
using var sourceStream = File.OpenRead("data.jsonl");
var extractor = new JsonLineExtractor<Person>(sourceStream, extractorLogger);

var transformer = new PassThroughTransformer<Person>();

using var destStream = File.Create("output.json");
var loader = new JsonSingleStreamLoader<Person>(destStream, loaderLogger);

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        cancellationToken
    ),
    cancellationToken
);
```

> **Note:** `PassThroughTransformer<T>` is a planned class tracked by [Chris-Wolfgang/ETL-Transformers#1](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/1). Until it ships, write a small pass-through subclass of `TransformerBase<T, T, TProgress>` whose `TransformWorkerAsync` iterates the input and yields each item after calling `IncrementCurrentItemCount()`.


## Chaining Transformers

Prefer many small, focused transformers over one large transformer:

1. **Smaller transformers are easier to test.** Each one has a narrow input/output contract and a single responsibility, which keeps unit tests small and focused.
2. **Smaller transformers are more reusable.** A transformer that validates an `Address`, for example, can be reused in any ETL that produces an `Address` -- you just add conversion transformers on either side to adapt from the ERP/CRM/etc. format to `Address` and back. Composing a complex transformation from smaller pieces across projects is much easier than reusing a monolithic transformer.

When several transformers need to be chained together, use `MultistepTransformer` (from `Wolfgang.Etl.Transformers`) rather than nesting `TransformAsync` calls manually. `MultistepTransformer` takes an ordered collection of transformers where:

- The `TSource` of the `MultistepTransformer` matches the input of the first transformer.
- The output of transformer N matches the input of transformer N+1.
- The `TDestination` of the `MultistepTransformer` matches the output of the last transformer.

```csharp
var transformer = new MultistepTransformer<SourcePerson, DestinationContact, TransformerProgressReport>
(
    new ITransformAsync<object, object>[]
    {
        new CleanDataTransformer(),    // SourcePerson   -> CleanPerson
        new EnrichTransformer(),       // CleanPerson    -> EnrichedPerson
        new MapToContactTransformer(), // EnrichedPerson -> DestinationContact
    }
);

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        cancellationToken
    ),
    cancellationToken
);
```

This gives you a single transformer object you can hand to the loader, while still keeping each step independently testable, reusable, and composable.

> **Note:** `MultistepTransformer` is a planned class tracked by [Chris-Wolfgang/ETL-Transformers#2](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/2). Until it ships, chain transformers by nesting `TransformAsync` calls manually, with the innermost call receiving the extractor's enumerable.


## Interchangeability

Because all extractors implement the same interfaces, you can swap data sources without changing the rest of the pipeline:

```csharp
// Read from JSONL
var extractor = new JsonLineExtractor<SourcePerson>(jsonStream, logger);

// Or read from an XML file -- same TRecord, different source format
var extractor = new XmlSingleStreamExtractor<SourcePerson>(xmlStream, xmlOptions, logger);

// Or read from a database -- same TRecord, totally different source model
var extractor = new DbExtractor<SourcePerson>
(
    connection,
    "SELECT first_name, last_name, date_of_birth FROM persons",
    logger: logger
);
```

The transformer and loader do not care where the data came from -- file, stream, or database. Similarly, you can swap the loader:

```csharp
// Write to JSONL
var loader = new JsonLineLoader<DestinationContact>(stream, logger);

// Or write to an XML file -- same TDestination, different destination format
var loader = new XmlSingleStreamLoader<DestinationContact>(stream, xmlOptions, logger);

// Or write to a database -- same TDestination, totally different destination model
var loader = new DbLoader<DestinationContact>
(
    connection,
    WriteMode.Insert,
    logger: logger
);
```


## Error Handling

Each component is responsible for handling errors within its domain:
- The **extractor** handles source read failures (file not found, network errors, malformed data)
- The **transformer** handles conversion failures (invalid values, missing fields)
- The **loader** handles write failures (disk full, permissions, API errors)

Unhandled exceptions propagate up through the async enumeration chain and surface at the `LoadAsync` call site.


## Testing the Pipeline

Test the pipeline at two levels: each component in isolation (unit tests), and the components wired together (integration tests).

### Unit testing each component

Each extractor, transformer, and loader should have its own unit-test class that verifies it in isolation from the rest of the pipeline. Use the contract test base classes from `Wolfgang.Etl.TestKit.Xunit` -- inheriting from them gives you 20+ test cases verifying that your component honors the contract shared by every component built on the Abstractions base classes.

```csharp
public class PersonToContactTransformerTests
    : TransformerBaseContractTests<PersonToContactTransformer, SourcePerson, Report>
{
    protected override PersonToContactTransformer CreateSut(int itemCount)
        => new();

    protected override IReadOnlyList<SourcePerson> CreateExpectedItems()
        => Enumerable.Range(0, 5)
            .Select(i => new SourcePerson { first_name = $"F{i}", last_name = $"L{i}" })
            .ToList();

    protected override PersonToContactTransformer CreateSutWithTimer(IProgressTimer timer)
        => new(timer);
}
```

That's the entire test class. The base class runs every contract assertion against it.

For the full TestKit API -- contract test base classes, test doubles (`TestExtractor<T>`, `TestLoader<T>`, `TestTransformer<T>`), and the timer injection pattern needed for progress tests -- see [TestKit](TestKit).

### Integration testing the pipeline

Unit tests verify each component behaves correctly on its own. Integration tests verify the components actually wire together and produce the right end-to-end result:

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

Keep integration tests small -- a handful of end-to-end cases that confirm the pipeline is wired correctly. The heavy lifting (boundary conditions, cancellation, progress, skip/max counts) is already covered by the component-level contract tests.


## See Also

- [Extractor](Extractor) -- API reference
- [Transformer](Transformer) -- API reference
- [Loader](Loader) -- API reference
- [TestKit](TestKit) -- test doubles and contract tests
- [Building an Extractor](Building-an-Extractor) -- step-by-step extractor guide
- [Building a Loader](Building-a-Loader) -- step-by-step loader guide
