# Database to File

Batch export: stream rows from a database query and write them to a CSV file, keeping memory flat no matter how many rows the query returns.

| Stage | Component | Package |
|-------|-----------|---------|
| Extract | `DbExtractor<Customer>` | `Wolfgang.Etl.DbClient` |
| Transform | `SelectTransformer<Customer, CustomerExport>` | `Wolfgang.Etl.Transformers` |
| Load | `CsvLoader<CustomerExport>` | `Wolfgang.Etl.Csv` |

```bash
dotnet add package Wolfgang.Etl.DbClient
dotnet add package Wolfgang.Etl.Transformers
dotnet add package Wolfgang.Etl.Csv
dotnet add package Microsoft.Data.SqlClient
```

`DbExtractor<T>` works with any ADO.NET provider; this recipe uses SQL Server through `Microsoft.Data.SqlClient`.


## Source and destination shapes

```csharp
// What the extractor yields - one property per selected column (matched case-insensitively)
public record Customer
{
    public int Id { get; init; }
    public string? FirstName { get; init; }
    public string? LastName { get; init; }
    public string? Email { get; init; }
    public DateTime CreatedAt { get; init; }
}



// What the loader writes - one CSV column per property
public record CustomerExport
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";
    public string CreatedOn { get; init; } = "";
}
```


## The pipeline

```csharp
using System.Globalization;
using Microsoft.Data.SqlClient;
using Wolfgang.Etl.Csv;
using Wolfgang.Etl.DbClient;
using Wolfgang.Etl.Transformers;

public static async Task ExportCustomersAsync
(
    string connectionString,
    string path,
    CancellationToken token
)
{
    // Extract: the caller owns the connection - DbExtractor never opens or closes it
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync(token);

    var extractor = new DbExtractor<Customer>
    (
        connection,
        "SELECT Id, FirstName, LastName, Email, CreatedAt FROM dbo.Customers WHERE IsActive = 1"
    );

    // Transform: flatten and format for the export
    var transformer = new SelectTransformer<Customer, CustomerExport>
    (
        customer => new CustomerExport
        {
            Id = customer.Id,
            Name = $"{customer.FirstName} {customer.LastName}".Trim(),
            Email = customer.Email ?? "",
            CreatedOn = customer.CreatedAt.ToString("yyyy-MM-dd", CultureInfo.InvariantCulture),
        }
    );

    // Load: CsvLoader writes through a StreamWriter
    await using var writer = new StreamWriter(path);
    var loader = new CsvLoader<CustomerExport>(writer);

    await loader.LoadAsync
    (
        transformer.TransformAsync(extractor.ExtractAsync(token)),
        token
    );
}
```

Rows stream from the data reader through the transformer into the file one at a time; nothing collects the result set in memory.


## Variations

- **JSON instead of CSV.** Replace the loader with `JsonSingleStreamLoader<CustomerExport>` (one JSON array) or `JsonLineLoader<CustomerExport>` (one object per line) from `Wolfgang.Etl.Json`; both take a `Stream` rather than a `StreamWriter`. [Your First ETL](../getting-started/your-first-etl.md) shows the JSON version end to end.
- **No reshaping needed.** If the file should have exactly the extractor's columns, use `PassThroughTransformer<Customer>` (also in `Wolfgang.Etl.Transformers`) and load `Customer` directly.
- **Query parameters.** Use the `DbExtractor<T>` constructor that takes an `IDictionary<string, object>` of parameters rather than concatenating values into the SQL text.
- **A compressed export.** Wrap the output file stream in a `GZipStream` and give the `StreamWriter` that stream; see [Reading and Writing Compressed Files and Streams](../guides/compressed-streams.md).


## Related

- [DbClient](../libraries/dbclient.md) - connection ownership, options and parameter handling
- [Pipeline Composition](../guides/pipeline-composition.md) - how the three stages are wired
- [Performance](../guides/performance.md) - buffer sizes and batching
