# Wolfgang.Etl.DbClient

ADO.NET database extractor and loader using Dapper for the ETL Framework.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.DbClient` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.DbClient) |
| **Dependencies** | Wolfgang.Etl.Abstractions, Dapper, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-DbClient](https://github.com/Chris-Wolfgang/ETL-DbClient) |

## Overview

Wolfgang.Etl.DbClient provides database extraction and loading via ADO.NET and Dapper. Records are streamed as `IAsyncEnumerable<T>` using Dapper's `QueryUnbufferedAsync`, so result sets are not buffered in memory. The loader executes one command per record within a transaction, supporting both caller-managed and auto-managed transaction modes.

## DbExtractor

`DbExtractor<TRecord, TProgress>` inherits `ExtractorBase<TRecord, TProgress>`.

**Type constraints:** `TRecord : notnull` and `TProgress : notnull`.

### Constructors

```csharp
// SQL command:
public DbExtractor
(
    DbConnection connection,
    string commandText,
    DbTransaction? transaction = null,
    ILogger<DbExtractor<TRecord, TProgress>>? logger = null
);

// Parameterized SQL command:
public DbExtractor
(
    DbConnection connection,
    string commandText,
    Dictionary<string, object> parameters,
    DbTransaction? transaction = null,
    ILogger<DbExtractor<TRecord, TProgress>>? logger = null
);

// Auto-generated SELECT from [Table] and [Column] attributes:
public DbExtractor
(
    DbConnection connection,
    DbTransaction? transaction = null,
    ILogger<DbExtractor<TRecord, TProgress>>? logger = null
);
```

### Properties

| Property | Type | Description |
|---|---|---|
| `CommandText` | `string` | The SQL command text being executed (read-only). |

### Ownership semantics

The caller owns the `DbConnection` lifetime. The extractor does not open, close, or dispose it. The connection must be open before calling `ExtractAsync`. An optional `DbTransaction` can be provided for isolation level control; the extractor never commits or rolls back.

### Basic extraction example

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cancellationToken);

var extractor = new DbExtractor<CustomerRecord, DbReport>
(
    connection,
    "SELECT Id, FirstName, LastName FROM Customers WHERE Active = @Active",
    new Dictionary<string, object> { ["Active"] = true }
);

await foreach (var customer in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine($"{customer.FirstName} {customer.LastName}");
}
```

### Auto-generated SELECT

When no command text is provided, the extractor generates a SELECT statement from `[Table]` and `[Column]` attributes:

```csharp
[Table("Customers")]
public class CustomerRecord
{
    [Key]
    [Column("customer_id")]
    public int Id { get; set; }

    [Column("first_name")]
    public string FirstName { get; set; }

    [Column("last_name")]
    public string LastName { get; set; }

    [NotMapped]
    public string FullName => $"{FirstName} {LastName}";
}

// Generates: SELECT customer_id AS Id, first_name AS FirstName, last_name AS LastName FROM Customers
var extractor = new DbExtractor<CustomerRecord, DbReport>(connection);
```

## DbLoader

`DbLoader<TRecord, TProgress>` inherits `LoaderBase<TRecord, TProgress>`.

**Type constraints:** `TRecord : notnull` and `TProgress : notnull`.

### Constructors

```csharp
// Custom SQL command:
public DbLoader
(
    DbConnection connection,
    string commandText,
    DbTransaction? transaction = null,
    ILogger<DbLoader<TRecord, TProgress>>? logger = null
);

// Auto-generated INSERT or UPDATE from attributes:
public DbLoader
(
    DbConnection connection,
    WriteMode writeMode,
    DbTransaction? transaction = null,
    ILogger<DbLoader<TRecord, TProgress>>? logger = null
);
```

### Properties

| Property | Type | Description |
|---|---|---|
| `CommandText` | `string` | The SQL command text being executed per record (read-only). |

### Transaction modes

| Mode | How to activate | Behavior |
|---|---|---|
| **Caller-managed** | Pass a `DbTransaction` to the constructor | The loader uses the transaction but never commits or rolls back. The caller is responsible for the transaction lifetime. |
| **Auto-managed** | Pass `null` for the transaction parameter | The loader creates its own transaction, commits on success, and rolls back on exception. |

### Insert loading example

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cancellationToken);

var loader = new DbLoader<CustomerRecord, DbReport>
(
    connection,
    WriteMode.Insert
);
// Generates: INSERT INTO Customers (first_name, last_name) VALUES (@FirstName, @LastName)

await loader.LoadAsync(customerStream, cancellationToken);
```

### Update loading example

```csharp
var loader = new DbLoader<CustomerRecord, DbReport>
(
    connection,
    WriteMode.Update
);
// Generates: UPDATE Customers SET first_name = @FirstName, last_name = @LastName WHERE customer_id = @Id

await loader.LoadAsync(customerStream, cancellationToken);
```

### Caller-managed transaction

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cancellationToken);
await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

var loader = new DbLoader<CustomerRecord, DbReport>
(
    connection,
    "INSERT INTO Customers (first_name, last_name) VALUES (@FirstName, @LastName)",
    transaction
);

try
{
    await loader.LoadAsync(customerStream, cancellationToken);
    await transaction.CommitAsync(cancellationToken);
}
catch
{
    await transaction.RollbackAsync(cancellationToken);
    throw;
}
```

## WriteMode enum

| Value | Description |
|---|---|
| `Insert` | Generates an INSERT statement. All non-identity-key columns are included. |
| `Update` | Generates an UPDATE statement. Requires at least one `[Key]` property for the WHERE clause. |

## DbReport

Extends `Report` with database-specific progress data.

| Property | Type | Description |
|---|---|---|
| `CurrentCount` | `int` | Number of records processed (inherited). |
| `CurrentSkippedItemCount` | `int` | Number of records skipped. |
| `CommandText` | `string` | The SQL command text being executed. |
| `ElapsedMilliseconds` | `long` | Wall-clock time since the operation started. |

## Attribute usage for auto-generated SQL

The `DbCommandBuilder` uses standard `System.ComponentModel.DataAnnotations` and `System.ComponentModel.DataAnnotations.Schema` attributes:

| Attribute | Purpose |
|---|---|
| `[Table("name")]` | Required. Specifies the database table name. |
| `[Column("name")]` | Maps a property to a column with a different name. Without it, the property name is used. |
| `[Key]` | Marks the property as a primary key. Used in UPDATE WHERE clauses and excluded from INSERT when combined with `[DatabaseGenerated(Identity)]`. |
| `[DatabaseGenerated(Identity)]` | Combined with `[Key]`, excludes the column from INSERT (auto-increment). Non-identity keys are included since they are user-assigned. |
| `[NotMapped]` | Excludes the property from all generated SQL. |

### Auto-generated SQL behavior

**BuildSelect**: Generates `SELECT col AS Prop, ... FROM Table`. If no `[Column]` attributes exist, generates `SELECT * FROM Table`.

**BuildInsert**: Generates `INSERT INTO Table (col, ...) VALUES (@Prop, ...)`. Excludes `[Key] + [DatabaseGenerated(Identity)]` columns. Non-identity keys are included.

**BuildUpdate**: Generates `UPDATE Table SET col = @Prop, ... WHERE keyCol = @KeyProp`. Requires at least one `[Key]` property. Throws `InvalidOperationException` if none exist.

## ColumnAttributeTypeMapper

Dapper normally maps result set columns by property name only. The `ColumnAttributeTypeMapper` (registered automatically in the static constructor) enables `[Column]` attribute support. It checks `[Column("name")]` first, then falls back to Dapper's default case-insensitive name matching.

## Connection lifecycle

The caller is fully responsible for the `DbConnection`:

1. Create and open the connection before passing it to the extractor or loader.
2. The extractor/loader uses the connection but does not open, close, or dispose it.
3. Close and dispose the connection after extraction/loading is complete.

This design supports connection pooling, ambient transactions, and multi-operation pipelines on the same connection.
