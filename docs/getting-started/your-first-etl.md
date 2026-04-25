# Your First ETL

This guide builds a complete end-to-end ETL pipeline using libraries that already exist — no custom extractor or loader required. The scenario: read employee records from a SQL Server database table and write them to a single JSON file.

By the end you will have a runnable program that demonstrates the framework's three-stage pipeline shape: extractor → transformer → loader.


## Prerequisites

- A supported [.NET SDK](https://dotnet.microsoft.com/download). The framework's `Wolfgang.Etl.Abstractions` package runs on .NET Framework 4.6.2+, .NET Core 3.1, and .NET 5+ (or any runtime supporting `netstandard2.0`/`netstandard2.1`). Individual library packages may have a narrower matrix — see [Installation](installation.md) and the relevant [Libraries](../libraries/index.md) page.
- A SQL Server instance with an `Employees` table (any provider works — `Microsoft.Data.SqlClient` is the default in this example)

Two NuGet packages do all the heavy lifting:

```bash
dotnet add package Wolfgang.Etl.DbClient
dotnet add package Wolfgang.Etl.Json
dotnet add package Microsoft.Data.SqlClient
```

`Wolfgang.Etl.Abstractions` is pulled in automatically as a transitive dependency.


## Step 1: Define your record type

The record type matches the columns of the `Employees` table you want to extract:

```csharp
public record Employee
{
    public int Id { get; set; }
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? Email { get; set; }
    public DateTime HireDate { get; set; }
}
```

Property names map to columns by case-insensitive match. Use `record` for value equality and a clean `ToString()` for free.


## Step 2: Build a pass-through transformer

Every ETL has a transformer stage — even when you do not need to convert the data. For this scenario the source and destination types are the same (`Employee` → `Employee`), so the transformer is a thin pass-through.

```csharp
public class EmployeePassThrough : TransformerBase<Employee, Employee, Report>
{
    protected override async IAsyncEnumerable<Employee> TransformWorkerAsync
    (
        IAsyncEnumerable<Employee> items,
        [EnumeratorCancellation] CancellationToken token
    )
    {
        await foreach (var employee in items.WithCancellation(token).ConfigureAwait(false))
        {
            IncrementCurrentItemCount();
            yield return employee;
        }
    }



    protected override Report CreateProgressReport() => new(CurrentItemCount);
}
```

!!! note "PassThroughTransformer&lt;T&gt; is a planned class"
    Once `Wolfgang.Etl.Transformers` ships, this will collapse to a single line: `var transformer = new PassThroughTransformer<Employee>();`. Tracked by [Chris-Wolfgang/ETL-Transformers#1](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/1).


## Step 3: Wire and run the pipeline

```csharp
const string connectionString = "Server=.;Database=YourDb;Integrated Security=true;TrustServerCertificate=true";
using var cts = new CancellationTokenSource();

// Open the connection -- the DbExtractor never opens or closes it
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cts.Token);

// Extractor: SQL Server -> Employee
var extractor = new DbExtractor<Employee, DbReport>
(
    connection,
    "SELECT id, first_name, last_name, email, hire_date FROM employees",
    logger: extractorLogger
);

// Transformer: pass-through
var transformer = new EmployeePassThrough();

// Loader: Employee -> JSON array file
await using var fileStream = File.Create("employees.json");
var loader = new JsonSingleStreamLoader<Employee>(fileStream, loaderLogger);

// Wire and run
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

That is the entire pipeline. The loader pulls items through the transformer from the extractor on demand — no buffering, no intermediate collections, memory usage stays constant regardless of how many rows the table has.

The output `employees.json` is a JSON array:

```json
[{"Id":1,"FirstName":"Alice","LastName":"Smith","Email":"alice@example.com","HireDate":"2020-01-15T00:00:00"},{"Id":2,"FirstName":"Bob","LastName":"Jones","Email":"bob@example.com","HireDate":"2019-03-22T00:00:00"}]
```


## Variation: choose the output format at runtime

The extractor and transformer do not care which loader is on the other end of the pipeline — every loader for `Employee` honors the same contract. That means you can decide the output format at run time and assign the corresponding loader to a single variable.

This variation extends the pipeline above to accept a `--format json|csv` command-line argument and pick the matching loader. Steps 1 and 2 are unchanged; only the wiring changes.

### Add a tiny CSV loader

`Wolfgang.Etl.Json` is on NuGet, but a CSV library is not yet shipped. Drop in a small inline loader that subclasses `LoaderBase<Employee, Report>` and writes one record per line — same pattern as [Your First Loader](your-first-loader.md).

```csharp
public class EmployeeCsvLoader : LoaderBase<Employee, Report>
{
    private readonly StreamWriter _writer;


    public EmployeeCsvLoader(Stream destination)
    {
        _writer = new StreamWriter(destination);
    }


    protected override async Task LoadWorkerAsync
    (
        IAsyncEnumerable<Employee> items,
        CancellationToken token
    )
    {
        await _writer.WriteLineAsync("Id,FirstName,LastName,Email,HireDate").ConfigureAwait(false);

        await foreach (var e in items.WithCancellation(token).ConfigureAwait(false))
        {
            await _writer
                .WriteLineAsync($"{e.Id},{e.FirstName},{e.LastName},{e.Email},{e.HireDate:O}")
                .ConfigureAwait(false);
            IncrementCurrentItemCount();
        }

        await _writer.FlushAsync().ConfigureAwait(false);
    }


    protected override Report CreateProgressReport() => new(CurrentItemCount);
}
```

!!! note "CsvSingleStreamLoader&lt;T&gt; is a planned class"
    Once `Wolfgang.Etl.Csv` ships, the inline loader above can be replaced with `var loader = new CsvSingleStreamLoader<Employee>(fileStream);`. Field escaping, quoting, and culture-aware date formatting will be handled by the library — the snippet here is intentionally minimal so it fits in a tutorial.

### Pick the loader at run time

Type the loader variable as `ILoadWithCancellationAsync<Employee>` and assign whichever concrete instance matches the requested format. The rest of the pipeline does not change.

```csharp
// args[0] is "json" or "csv"
var format = args.Length > 0 ? args[0] : "json";
var path = $"employees.{format}";

await using var fileStream = File.Create(path);

ILoadWithCancellationAsync<Employee> loader = format switch
{
    "json" => new JsonSingleStreamLoader<Employee>(fileStream, loaderLogger),
    "csv"  => new EmployeeCsvLoader(fileStream),
    _      => throw new ArgumentException($"Unknown format: {format}")
};

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

`ILoadWithCancellationAsync<Employee>` is the framework interface every cancellation-aware loader implements (see [Architecture](../architecture.md#interfaces) for the full diamond). Because the variable is typed as the interface — not as `JsonSingleStreamLoader<Employee>` or `EmployeeCsvLoader` — the same `LoadAsync` call drives whichever loader was selected.

The same swap works in the other direction: keep the loader fixed and swap the extractor at run time when the source format depends on configuration. Any extractor of `TSource` can replace any other extractor of the same `TSource`.


## What's next

You just built an ETL using two pre-built libraries. When you need to do more:

- **[Your First Extractor](your-first-extractor.md)** — build a custom extractor when no library exists for your source format
- **[Your First Transformer](your-first-transformer.md)** — build a custom transformer for your domain-specific mapping logic
- **[Your First Loader](your-first-loader.md)** — build a custom loader when no library exists for your destination format
- **[Pipeline Composition](../guides/pipeline-composition.md)** — variations on the basic pattern: chaining multiple transformers, cancellation, progress reporting, error handling, the full worked example
- **[Libraries](../libraries/index.md)** — what extractors and loaders are available out of the box (FixedWidth, DbClient, XML, JSON, and more)
- **[Architecture](../architecture.md)** — the layered design that makes components from different libraries swappable
