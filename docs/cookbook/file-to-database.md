# File to Database

Batch import: read a JSON Lines file, map each line to the shape of a database table, and bulk-insert the rows into SQL Server.

| Stage | Component | Package |
|-------|-----------|---------|
| Extract | `JsonLineExtractor<OrderLine>` | `Wolfgang.Etl.Json` |
| Transform | `SelectTransformer<OrderLine, OrderRow>` | `Wolfgang.Etl.Transformers` |
| Load | `SqlBulkCopyLoader<OrderRow>` | `Wolfgang.Etl.SqlBulkCopy` |

```bash
dotnet add package Wolfgang.Etl.Json
dotnet add package Wolfgang.Etl.Transformers
dotnet add package Wolfgang.Etl.SqlBulkCopy
```

`Wolfgang.Etl.SqlBulkCopy` brings in `Microsoft.Data.SqlClient`; `Wolfgang.Etl.Abstractions` comes in transitively with all three.


## Source and destination shapes

The file holds one JSON object per line:

```json
{"orderId":"A-1001","sku":"WIDGET-01","quantity":3,"unitPrice":4.25,"orderedAt":"2026-10-01T09:30:00Z"}
{"orderId":"A-1002","sku":"GADGET-07","quantity":1,"unitPrice":19.99,"orderedAt":"2026-10-01T09:41:12Z"}
```

```csharp
// What the extractor yields - mirrors the JSON
public record OrderLine
{
    public string OrderId { get; init; } = "";
    public string Sku { get; init; } = "";
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
    public DateTimeOffset OrderedAt { get; init; }
}



// What the loader writes - mirrors the dbo.OrderLines table
public record OrderRow
{
    public string OrderId { get; init; } = "";
    public string Sku { get; init; } = "";
    public int Quantity { get; init; }
    public decimal LineTotal { get; init; }
    public DateTime OrderedAtUtc { get; init; }
}
```


## The pipeline

```csharp
using System.Text.Json;
using Microsoft.Data.SqlClient;
using Wolfgang.Etl.Json;
using Wolfgang.Etl.SqlBulkCopy;
using Wolfgang.Etl.Transformers;

public static async Task ImportOrdersAsync
(
    string path,
    string connectionString,
    CancellationToken token
)
{
    // Extract: one OrderLine per line of the file
    await using var file = File.OpenRead(path);
    var extractor = new JsonLineExtractor<OrderLine>
    (
        file,
        new JsonSerializerOptions(JsonSerializerDefaults.Web)
    );

    // Transform: reshape to the table's columns
    var transformer = new SelectTransformer<OrderLine, OrderRow>
    (
        line => new OrderRow
        {
            OrderId = line.OrderId,
            Sku = line.Sku,
            Quantity = line.Quantity,
            LineTotal = line.Quantity * line.UnitPrice,
            OrderedAtUtc = line.OrderedAt.UtcDateTime,
        }
    );

    // Load: SqlBulkCopy into dbo.OrderLines (the loader does not open or close the connection)
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync(token);

    var loader = new SqlBulkCopyLoader<OrderRow>
    (
        connection,
        new SqlBulkCopyLoaderOptions<OrderRow>
        {
            DestinationSchemaName = "dbo",
            DestinationTableName = "OrderLines",
            BatchSize = 5_000,
        }
    );

    await loader.LoadAsync
    (
        transformer.TransformAsync(extractor.ExtractAsync(token)),
        token
    );
}
```

The loader pulls from the transformer, which pulls from the extractor, so the file is never read into memory as a whole; the loader holds at most one bulk-copy batch at a time.

`JsonSerializerDefaults.Web` makes property-name matching case-insensitive, which is what the camelCase sample file needs. Pass whatever `JsonSerializerOptions` your file needs.


## Variations

- **Another file format.** Swap the extractor; the transformer and loader do not change. `CsvExtractor<T>` (`Wolfgang.Etl.Csv`, takes a `StreamReader`), `XmlSingleStreamExtractor<T>` (`Wolfgang.Etl.Xml`) and `FixedWidthExtractor<T>` (`Wolfgang.Etl.FixedWidth`) all yield items from a stream the same way.
- **A database other than SQL Server.** `SqlBulkCopyLoader<T>` is SQL Server only. For any ADO.NET provider use `DbLoader<T>` from `Wolfgang.Etl.DbClient` with `WriteMode.Insert`; it maps the record to a table through `[Table]` / `[Column]` attributes and batches the inserts:

    ```csharp
    var loader = new DbLoader<OrderRow>
    (
        connection,
        WriteMode.Insert,
        new DbLoaderOptions { InsertBatchSize = 500 }
    );
    ```

- **A compressed file.** Wrap the file stream in a `GZipStream` before handing it to the extractor; see [Reading and Writing Compressed Files and Streams](../guides/compressed-streams.md).
- **Bad lines.** Decide whether one malformed line should stop the import or be skipped; see [Error Handling](../guides/error-handling.md).


## Related

- [Pipeline Composition](../guides/pipeline-composition.md) - how the three stages are wired
- [Progress Reporting](../guides/progress-reporting.md) - reporting rows imported while the load runs
- [Libraries](../libraries/index.md) - which package covers which format
