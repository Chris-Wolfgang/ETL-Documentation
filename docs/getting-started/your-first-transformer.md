# Your First Transformer

This guide walks through building a custom transformer from scratch. The scenario: convert `RawOrderLine` records (text fields from a flat file) into `OrderLine` records with proper types and a computed total.

By the end you will have a fully tested transformer that demonstrates the typical pattern: type conversion, validation, and a computed field.


## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.26.1" />
```


## Interfaces vs. the Base Class

There are two ways to build a transformer:

1. **Inherit from `TransformerBase<TSource, TDestination, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, configuration validation, progress timer setup) and leaves you to implement a single `TransformWorkerAsync` method. This guide focuses on this path.
2. **Implement one of the transformer interfaces directly** — for full control. You decide exactly how every piece of the transformer behaves, at the cost of implementing everything yourself.

See [Architecture § Interfaces](../architecture.md#interfaces) for the four-interface diamond shared across all three stages.


## Step 1: Define Your Source and Destination Types

A transformer maps `TSource` to `TDestination`. Define both:

```csharp
// Source: what an extractor (e.g. FixedWidthExtractor or JsonLineExtractor) yields.
// Strings everywhere because the source is a text file.
public record RawOrderLine
{
    public string? OrderId { get; set; }
    public string? ProductCode { get; set; }
    public string? Quantity { get; set; }
    public string? UnitPrice { get; set; }
    public string? Currency { get; set; }
}

// Destination: what the loader writes.
// Properly typed, plus a computed Total field.
public record OrderLine
{
    public string OrderId { get; set; } = string.Empty;
    public string ProductCode { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public string Currency { get; set; } = string.Empty;
    public decimal Total { get; set; }
}
```


## Step 2: Define Your Progress Report

Most transformers can use the `Report` base class from Abstractions directly — there is rarely transformer-specific progress data worth adding. Use a custom report only when you need something the base class does not provide (e.g. a count of records that failed validation):

```csharp
using Wolfgang.Etl.Abstractions;

public record OrderLineReport : Report
{
    public OrderLineReport(int currentItemCount, int currentSkippedItemCount)
        : base(currentItemCount)
    {
        CurrentSkippedItemCount = currentSkippedItemCount;
    }



    public int CurrentSkippedItemCount { get; }
}
```


## Step 3: Create the Transformer Class

Inherit from `TransformerBase<TSource, TDestination, TProgress>` where:

- `TSource` is the type your transformer consumes (must be `notnull`)
- `TDestination` is the type your transformer yields (must be `notnull`)
- `TProgress` is your progress report type

```csharp
public class OrderLineTransformer
    : TransformerBase<RawOrderLine, OrderLine, OrderLineReport>
{
    private readonly ILogger _logger;
```

| Field | Purpose |
|-------|---------|
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |

The stage-level configuration (`SkipItemCount`, `MaximumItemCount`, `ReportingInterval`, error policy) arrives in a `TransformerOptions` record passed to the base constructor.


## Step 4: Constructors

One public constructor is enough; it passes the `TransformerOptions` to the base class, which validates them and throws `ArgumentOutOfRangeException` for out-of-range values (`null` means "use the defaults"):

```csharp
    public OrderLineTransformer
    (
        ILogger<OrderLineTransformer>? logger = null,
        TransformerOptions? options = null
    )
        : base(options)
    {
        _logger = (ILogger?)logger ?? NullLogger.Instance;
    }
```

Transformers typically have simpler constructors than extractors or loaders because they have no stream or connection to manage — the input arrives via `TransformAsync`'s parameter.


## Step 5: Implement TransformWorkerAsync

This is the core of your transformer. It is an `async` iterator method that consumes the input enumerable and yields transformed items one at a time:

```csharp
    protected override async IAsyncEnumerable<OrderLine> TransformWorkerAsync
    (
        IAsyncEnumerable<RawOrderLine> items,
        [EnumeratorCancellation] CancellationToken token
    )
    {
        var skipBudget = SkipItemCount;

        await foreach (var raw in items.WithCancellation(token).ConfigureAwait(false))
        {
            token.ThrowIfCancellationRequested();

            // Handle SkipItemCount (a local budget, so invalid lines skipped below do not use it up)
            if (skipBudget > 0)
            {
                skipBudget--;
                IncrementCurrentSkippedItemCount();
                continue;
            }

            // Handle MaximumItemCount
            if (CurrentItemCount >= MaximumItemCount)
            {
                break;
            }

            // Type conversion + validation
            if (!int.TryParse(raw.Quantity, out var quantity) ||
                !decimal.TryParse(raw.UnitPrice, out var unitPrice))
            {
                _logger.LogWarning("Skipping line with invalid number: {Raw}", raw);
                IncrementCurrentSkippedItemCount();
                continue;
            }

            // Build the destination record (computed Total field)
            var line = new OrderLine
            {
                OrderId      = raw.OrderId      ?? string.Empty,
                ProductCode  = raw.ProductCode  ?? string.Empty,
                Quantity     = quantity,
                UnitPrice    = unitPrice,
                Currency     = raw.Currency     ?? string.Empty,
                Total        = quantity * unitPrice,
            };

            IncrementCurrentItemCount();
            yield return line;
        }
    }
```

### Critical Rules

1. **Call `IncrementCurrentItemCount()` exactly once per yielded item.** Not in helper methods, not conditionally — once, right before or after the `yield return`.
2. **Implement `SkipItemCount` and `MaximumItemCount` yourself.** The base class validates them but does not enforce them.
3. **Respect the `CancellationToken`.** Call `token.ThrowIfCancellationRequested()` in your loop.
4. **Use `[EnumeratorCancellation]` on the token parameter.** Required for `async IAsyncEnumerable` methods so that `WithCancellation()` works correctly.
5. **Use `.ConfigureAwait(false)`** on all `await` calls in library code.
6. **Decide your error policy.** A transformer can throw on a bad record (the exception propagates), skip the record (as shown above), or substitute a default. See [Error Handling](../guides/error-handling.md) for the trade-offs.


## Step 6: CreateProgressReport

Identical pattern to extractors and loaders:

```csharp
    protected override OrderLineReport CreateProgressReport() =>
        new
        (
            CurrentItemCount,
            CurrentSkippedItemCount
        );
}
```

No timer plumbing is needed: the base class runs the production timer, and TestKit drives progress in tests through the base class (see [TestKit § Progress Timing in Tests](../reference/testkit.md#progress-timing-in-tests)). Earlier versions of this guide added an `IProgressTimer` field, an internal constructor, a `CreateProgressTimer` override and `InternalsVisibleTo`; leave them out of new transformers.


## Step 7: Write Tests

### The contract tests apply to same-type transformers

`TransformerBaseContractTests<TSut, TItem, TProgress>` from `Wolfgang.Etl.TestKit.Xunit` requires `TSut : TransformerBase<TItem, TItem, TProgress>` — it feeds items in and expects the same type out, so it fits validators, filters, enrichers and other `T` → `T` transformers. Its 48 tests then come from two factory methods, `CreateSut(int itemCount, int maximumItemCount, int skipItemCount, int reportingInterval)` and `CreateExpectedItems()`, exactly as in [Your First Extractor](your-first-extractor.md#inherit-the-contract-test-base-class).

`OrderLineTransformer` changes the type (`RawOrderLine` → `OrderLine`), so test it directly. The contract behaviors worth covering yourself are the ones your worker implements: `SkipItemCount`, `MaximumItemCount`, cancellation, and one `IncrementCurrentItemCount()` per yielded item.

### Add domain-specific tests

Verify the conversion rules and the configuration your worker honors:

```csharp
[Fact]
public async Task TransformAsync_when_quantity_invalid_skips_record()
{
    var raw = new[]
    {
        new RawOrderLine { OrderId = "1", Quantity = "not-a-number", UnitPrice = "10.00" },
        new RawOrderLine { OrderId = "2", Quantity = "3",            UnitPrice = "5.00"  },
    };

    var sut = new OrderLineTransformer(NullLogger<OrderLineTransformer>.Instance);

    var results = new List<OrderLine>();
    await foreach (var line in sut.TransformAsync(raw.ToAsyncEnumerable()))
    {
        results.Add(line);
    }

    Assert.Single(results);
    Assert.Equal("2", results[0].OrderId);
    Assert.Equal(15.00m, results[0].Total);
}



[Fact]
public async Task TransformAsync_when_SkipItemCount_is_1_skips_the_first_record()
{
    var raw = new[]
    {
        new RawOrderLine { OrderId = "1", Quantity = "1", UnitPrice = "1.00" },
        new RawOrderLine { OrderId = "2", Quantity = "2", UnitPrice = "1.00" },
    };

    var sut = new OrderLineTransformer(options: new TransformerOptions { SkipItemCount = 1 });

    var results = new List<OrderLine>();
    await foreach (var line in sut.TransformAsync(raw.ToAsyncEnumerable()))
    {
        results.Add(line);
    }

    Assert.Equal("2", Assert.Single(results).OrderId);
    Assert.Equal(1, sut.CurrentSkippedItemCount);
}
```

`raw.ToAsyncEnumerable()` turns the array into an `IAsyncEnumerable<RawOrderLine>`. It is not part of the framework packages: on .NET 10 and later it is built in (`System.Linq.AsyncEnumerable`); on older test targets add the `System.Linq.Async` package to the test project.


## Using Your Transformer

```csharp
// Wire into a pipeline (extractor and loader from any library)
var transformer = new OrderLineTransformer(transformerLogger);

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        cancellationToken
    ),
    cancellationToken
);

// With skip and maximum: configure them when the transformer is constructed
var paged = new OrderLineTransformer
(
    transformerLogger,
    new TransformerOptions { SkipItemCount = 100, MaximumItemCount = 1000 }
);

// With progress reporting
var progress = new Progress<OrderLineReport>(r =>
    Console.WriteLine($"Transformed {r.CurrentItemCount} items, {r.CurrentSkippedItemCount} skipped")
);

await loader.LoadAsync
(
    paged.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        progress,
        cancellationToken
    ),
    cancellationToken
);
```


## Checklist

- [ ] Inherit `TransformerBase<TSource, TDestination, TProgress>` (or `TransformerBase<TSource, TDestination>`) with `where TSource : notnull` and `where TDestination : notnull`
- [ ] Accept a `TransformerOptions?` (or a record derived from it) and pass it to `base(options)`
- [ ] Implement `TransformWorkerAsync` with `[EnumeratorCancellation]` on the token
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per yielded item
- [ ] Implement `SkipItemCount` logic in your worker method
- [ ] Implement `MaximumItemCount` logic in your worker method
- [ ] Respect `CancellationToken` (pass to async calls, call `ThrowIfCancellationRequested`)
- [ ] Same-type (`T` → `T`) transformer: test class inherits `TransformerBaseContractTests<TSut, TItem, TProgress>`, implements the four-argument `CreateSut`, and `CreateExpectedItems()` returns at least 5 items
- [ ] Type-changing transformer: direct tests for conversion rules, `SkipItemCount`, `MaximumItemCount` and cancellation
- [ ] `TSource` and `TDestination` use `record` types for value equality
- [ ] Add domain-specific tests for transformer-specific behavior (validation rules, computed fields, error policies)


## See Also

- [Your First Extractor](your-first-extractor.md) — build a custom extractor; many of the patterns here mirror that guide
- [Your First Loader](your-first-loader.md) — build a custom loader
- [Your First ETL](your-first-etl.md) — wire everything together end-to-end
- [Pipeline Composition](../guides/pipeline-composition.md) — chaining multiple transformers, inline transformations, [`PassThroughTransformer`](https://chris-wolfgang.github.io/ETL-Transformers/api/Wolfgang.Etl.Transformers.PassThroughTransformer-1.html) and chaining transformers with `Then(...)` / [`ChainTransformer`](https://chris-wolfgang.github.io/ETL-Transformers/api/Wolfgang.Etl.Transformers.ChainTransformer-3.html)
- [Custom Parsers & Converters](../guides/custom-parsers.md) — when your transformation logic involves library-specific extension points
- [Transformer reference](../reference/transformer.md) — formal API reference
