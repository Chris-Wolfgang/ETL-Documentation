# Wolfgang.Etl.DbClient

ADO.NET database extractor and loader.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.DbClient` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.DbClient) |
| **Dependencies** | Wolfgang.Etl.Abstractions, Dapper, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-DbClient](https://github.com/Chris-Wolfgang/ETL-DbClient) |

## What it does

Streams records from any ADO.NET-compatible database, and writes records back one command at a time inside a transaction. Works with SQL Server, PostgreSQL, SQLite, MySQL, Oracle — anything with a `DbConnection` provider.

Records can be bound via hand-written SQL, parameterized SQL, or auto-generated SELECT/INSERT from `[Table]` and `[Column]` attributes on your record type. The caller owns the connection and transaction lifetime; the library never opens, closes, or commits on its own.

## When to use it

- **Reading from a database** as the source of an ETL pipeline (any relational DB with ADO.NET support)
- **Writing to a database** as the destination — either row-by-row INSERT, or a `WriteMode` that builds UPSERT / MERGE logic
- When you want **transaction isolation control** — the loader respects a caller-supplied `DbTransaction`
- When you want **result-set streaming** rather than buffering the whole query result in memory

## When to use something else

- For bulk insert of millions of rows, use the database's native bulk-load API (e.g. `SqlBulkCopy` for SQL Server) rather than row-by-row `INSERT`. A future `Wolfgang.Etl.SqlBulkCopy` library will target this case.
- For schema migrations, use a migration tool (EF Core migrations, Flyway, DbUp) — this library is for data movement, not schema evolution.

## Quick example

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cancellationToken);

var extractor = new DbExtractor<CustomerRecord, DbReport>
(
    connection,
    "SELECT id, first_name, last_name FROM customers WHERE active = @active",
    new Dictionary<string, object> { ["active"] = true }
);

await foreach (var customer in extractor.ExtractAsync(cancellationToken))
{
    // ...
}
```

## More detail

- **API reference** — see the [ETL-DbClient README](https://github.com/Chris-Wolfgang/ETL-DbClient#readme) and XML doc comments on the public types
- **Examples** — see the [`examples/` folder](https://github.com/Chris-Wolfgang/ETL-DbClient/tree/main/examples) in the source repo
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-DbClient/releases)
