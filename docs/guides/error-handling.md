# Error Handling

This guide covers error-handling patterns that apply across the framework. For library-specific exception hierarchies and handling policies (e.g. FixedWidth's `MalformedLineHandling` enum), see each library's own README.


## Per-component responsibility

Each component in the pipeline is responsible for errors within its domain:

- The **extractor** handles source read failures — file not found, malformed data, encoding issues, network errors, database connection failures.
- The **transformer** handles conversion failures — invalid values, missing fields, business-rule violations.
- The **loader** handles write failures — disk full, permissions, authentication, field overflow, destination unavailability.

Unhandled exceptions propagate up through the async enumeration chain and surface at the `LoadAsync` call site:

```csharp
try
{
    await loader.LoadAsync
    (
        transformer.TransformAsync
        (
            extractor.ExtractAsync(cancellationToken),
            cancellationToken
        ),
        cancellationToken
    );
}
catch (OperationCanceledException)
{
    // Pipeline was cancelled cooperatively via the token.
    logger.LogInformation("Pipeline cancelled.");
}
catch (Exception ex)
{
    // Library-specific exception families are covered below.
    logger.LogError(ex, "Pipeline failed.");
    throw;
}
```

The outer `catch (Exception)` is a last-resort backstop. In practice you will catch library-specific exception types at this level — or higher, depending on whether you want the caller to decide how to respond. See [Library-specific error handling](#library-specific-error-handling) below for what each library throws.


## Cancellation

All pipeline components accept a `CancellationToken` for cooperative cancellation. The token is checked at each iteration boundary:

```csharp
using var cts = new CancellationTokenSource();

// Cancel after 5 minutes
cts.CancelAfter(TimeSpan.FromMinutes(5));

// Or cancel on user request
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;
    cts.Cancel();
};

try
{
    await loader.LoadAsync
    (
        transformer.TransformAsync
        (
            extractor.ExtractAsync(cts.Token),
            cts.Token
        ),
        cts.Token
    );
}
catch (OperationCanceledException)
{
    logger.LogInformation
    (
        "Pipeline cancelled after processing {Count} items.",
        extractor.CurrentItemCount
    );
}
```

Passing the same token to all three components ensures the entire pipeline shuts down promptly. The extractor checks the token before reading each record, the transformer checks before each transformation, and the loader checks before consuming each item.


## Errors in custom transformers

Transformers are responsible for handling conversion errors within their domain. A common pattern is to log and skip bad records:

```csharp
protected override async IAsyncEnumerable<OutputRecord> TransformWorkerAsync
(
    IAsyncEnumerable<InputRecord> items,
    [EnumeratorCancellation] CancellationToken token
)
{
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

        OutputRecord? output;
        try
        {
            output = ConvertRecord(item);
        }
        catch (FormatException ex)
        {
            _logger.LogWarning(ex, "Skipping record due to format error.");
            IncrementCurrentSkippedItemCount();
            continue;
        }

        IncrementCurrentItemCount();
        yield return output;
    }
}
```

Choose between skipping, substituting a default, or letting the exception propagate based on what "one bad record" means for your pipeline.


## Library-specific error handling

Each library throws its own exception families and may expose configurable handling policies. See each library's README for the full hierarchy, recipes for catching at the right level of specificity, and any enum options.

| Library | Typical exceptions | Configurable handling |
|---------|--------------------|----------------------|
| [FixedWidth](../libraries/fixedwidth.md) | `MalformedLineException` hierarchy (line-too-short, field conversion failures, field overflow) | `MalformedLineHandling` and `BlankLineHandling` enums let you pick between throwing, skipping, or returning a default per malformed / blank line |
| [JSON](../libraries/json.md) | `JsonException` from the underlying serializer on malformed JSON | Serializer-options flags control strictness (case sensitivity, null handling, comment tolerance) |
| [XML](../libraries/xml.md) | `XmlException` on malformed XML; `InvalidOperationException` on type-mapping failures | Reader-settings control DTD processing, whitespace, and validation |
| [DbClient](../libraries/dbclient.md) | Provider-specific `DbException` subclasses (`SqlException`, `NpgsqlException`, etc.) on database errors | Caller owns the connection and transaction; handle via normal ADO.NET patterns |

For per-library exception-catching recipes, see each library's README:

- [ETL-FixedWidth README](https://github.com/Chris-Wolfgang/ETL-FixedWidth#readme)
- [ETL-Json README](https://github.com/Chris-Wolfgang/ETL-Json#readme)
- [ETL-Xml README](https://github.com/Chris-Wolfgang/ETL-Xml#readme)
- [ETL-DbClient README](https://github.com/Chris-Wolfgang/ETL-DbClient#readme)


## See Also

- [Pipeline Composition](pipeline-composition.md) — wiring extractors, transformers, and loaders together
- [Custom Parsers & Converters](custom-parsers.md) — customizing parsing and serialization hooks (errors thrown inside your delegates surface here)
- [Progress Reporting](progress-reporting.md) — observing pipeline state during long-running operations
