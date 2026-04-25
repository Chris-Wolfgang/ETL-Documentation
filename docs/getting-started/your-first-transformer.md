# Your First Transformer

This guide walks through building a custom transformer from scratch. The scenario: convert `RawOrderLine` records (text fields from a flat file) into `OrderLine` records with proper types and a computed total.

By the end you will have a fully tested transformer that demonstrates the typical pattern: type conversion, validation, and a computed field.


## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```


## Interfaces vs. the Base Class

There are two ways to build a transformer:

1. **Inherit from `TransformerBase<TSource, TDestination, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `TransformWorkerAsync` method. This guide focuses on this path.
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
    private readonly IProgressTimer? _progressTimer;
    private int _progressTimerWired;
```

| Field | Purpose |
|-------|---------|
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |
| `_progressTimer` | Nullable timer for test injection (see [TestKit](../reference/testkit.md)) |
| `_progressTimerWired` | Guard flag (`int`, set atomically via `Interlocked.CompareExchange`) preventing duplicate timer event subscriptions |


## Step 4: Constructors

You need at least one public constructor plus one internal constructor for testing:

```csharp
    // Public: for production use
    public OrderLineTransformer(ILogger<OrderLineTransformer> logger)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }



    // Internal: for timer injection in tests
    internal OrderLineTransformer(ILogger logger, IProgressTimer timer)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _progressTimer = timer ?? throw new ArgumentNullException(nameof(timer));
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
        await foreach (var raw in items.WithCancellation(token).ConfigureAwait(false))
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
2. **Implement `SkipItemCount` and `MaximumItemCount` yourself.** The base class does not enforce them.
3. **Respect the `CancellationToken`.** Call `token.ThrowIfCancellationRequested()` in your loop.
4. **Use `[EnumeratorCancellation]` on the token parameter.** Required for `async IAsyncEnumerable` methods so that `WithCancellation()` works correctly.
5. **Use `.ConfigureAwait(false)`** on all `await` calls in library code.
6. **Decide your error policy.** A transformer can throw on a bad record (the exception propagates), skip the record (as shown above), or substitute a default. See [Error Handling](../guides/error-handling.md) for the trade-offs.


## Step 6: CreateProgressReport and CreateProgressTimer

Identical pattern to extractors and loaders:

```csharp
    protected override OrderLineReport CreateProgressReport() =>
        new
        (
            CurrentItemCount,
            CurrentSkippedItemCount
        );



    protected override IProgressTimer CreateProgressTimer(IProgress<OrderLineReport> progress)
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

The `Interlocked.CompareExchange` guard prevents duplicate event subscriptions atomically. The fall-through `return base.CreateProgressTimer(progress)` is required so the production path gets a working `SystemProgressTimer`. See [TestKit](../reference/testkit.md) for the full timer-injection pattern and rationale.


## Step 7: Expose Internals to the Test Project

Create `Properties/AssemblyInfo.cs` in your source project:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("YourProject.Tests.Unit")]
```


## Step 8: Write Tests

Inherit the contract test base class from `Wolfgang.Etl.TestKit.Xunit`:

```csharp
public class OrderLineTransformerTests
    : TransformerBaseContractTests<OrderLineTransformer, RawOrderLine, OrderLineReport>
{
    private static readonly IReadOnlyList<RawOrderLine> ExpectedItems = new List<RawOrderLine>
    {
        new() { OrderId = "1", ProductCode = "A", Quantity = "2", UnitPrice = "10.00", Currency = "USD" },
        new() { OrderId = "2", ProductCode = "B", Quantity = "1", UnitPrice = "5.50",  Currency = "USD" },
        new() { OrderId = "3", ProductCode = "C", Quantity = "5", UnitPrice = "1.25",  Currency = "USD" },
        new() { OrderId = "4", ProductCode = "D", Quantity = "3", UnitPrice = "2.00",  Currency = "USD" },
        new() { OrderId = "5", ProductCode = "E", Quantity = "7", UnitPrice = "0.75",  Currency = "USD" },
    };



    protected override OrderLineTransformer CreateSut(int itemCount) =>
        new(NullLogger<OrderLineTransformer>.Instance);



    protected override IReadOnlyList<RawOrderLine> CreateExpectedItems() => ExpectedItems;



    protected override OrderLineTransformer CreateSutWithTimer(IProgressTimer timer) =>
        new(NullLogger<OrderLineTransformer>.Instance, timer);
}
```

That's the entire test class. The contract base provides 20+ tests covering all four `TransformAsync` overloads, cancellation, skip/max enforcement, progress callbacks, empty-source handling, and counting correctness.

### Add domain-specific tests

On top of the contract tests, verify behavior unique to your transformer:

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
```


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

// With skip and maximum
transformer.SkipItemCount = 100;
transformer.MaximumItemCount = 1000;

// With progress reporting
var progress = new Progress<OrderLineReport>(r =>
    Console.WriteLine($"Transformed {r.CurrentItemCount} items, {r.CurrentSkippedItemCount} skipped")
);

await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        progress,
        cancellationToken
    ),
    cancellationToken
);
```


## Checklist

- [ ] Inherit `TransformerBase<TSource, TDestination, TProgress>` with `where TSource : notnull` and `where TDestination : notnull`
- [ ] Implement `TransformWorkerAsync` with `[EnumeratorCancellation]` on the token
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per yielded item
- [ ] Implement `SkipItemCount` logic in your worker method
- [ ] Implement `MaximumItemCount` logic in your worker method
- [ ] Respect `CancellationToken` (pass to async calls, call `ThrowIfCancellationRequested`)
- [ ] Add `_progressTimer` field, `_progressTimerWired` flag (`int`), and internal constructor
- [ ] Override `CreateProgressTimer` with `Interlocked.CompareExchange` guard and `base` fall-through
- [ ] Add `InternalsVisibleTo` in `Properties/AssemblyInfo.cs`
- [ ] Test class inherits `TransformerBaseContractTests<TSut, TItem, TProgress>`
- [ ] `CreateExpectedItems()` returns at least 5 items
- [ ] `TSource` and `TDestination` use `record` types for value equality
- [ ] Add domain-specific tests for transformer-specific behavior (validation rules, computed fields, error policies)


## See Also

- [Your First Extractor](your-first-extractor.md) — build a custom extractor; many of the patterns here mirror that guide
- [Your First Loader](your-first-loader.md) — build a custom loader
- [Your First ETL](your-first-etl.md) — wire everything together end-to-end
- [Pipeline Composition](../guides/pipeline-composition.md) — chaining multiple transformers, inline transformations, the full PassThroughTransformer / MultistepTransformer story
- [Custom Parsers & Converters](../guides/custom-parsers.md) — when your transformation logic involves library-specific extension points
- [Transformer reference](../reference/transformer.md) — formal API reference
