# Wolfgang.Etl.Json

JSON extractor and loader supporting JSONL, JSON arrays, and multi-stream for the ETL Framework.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.Json` |
| **Version** | 0.1.0 |
| **Dependencies** | Wolfgang.Etl.Abstractions 0.10.2, System.Text.Json, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-Json](https://github.com/Chris-Wolfgang/ETL-Json) |

## Overview

Wolfgang.Etl.Json provides extraction and loading for three JSON formats:

| Format | Extractor | Loader | Description |
|---|---|---|---|
| **JSONL / NDJSON** | `JsonLineExtractor<T>` | `JsonLineLoader<T>` | One JSON object per line. |
| **JSON Array** | `JsonSingleStreamExtractor<T>` | `JsonSingleStreamLoader<T>` | A single JSON array `[{...},{...}]`. |
| **Multi-Stream** | `JsonMultiStreamExtractor<T>` | `JsonMultiStreamLoader<T>` | One JSON object per stream. |

All classes use `System.Text.Json` for serialization and accept optional `JsonSerializerOptions` for customization.

**Type constraints:** `TRecord : notnull` for all classes (no `new()` constraint required, unlike XML).

## JsonLineExtractor

`JsonLineExtractor<TRecord>` inherits `ExtractorBase<TRecord, JsonReport>`.

Reads a stream line by line, deserializing each non-empty line as a JSON object. Blank lines are skipped with a warning. Compatible with both JSONL and NDJSON formats.

### Constructors

```csharp
// Basic:
public JsonLineExtractor
(
    Stream stream,
    ILogger<JsonLineExtractor<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonLineExtractor
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonLineExtractor<TRecord>> logger
);
```

### Example

```csharp
// Input (data.jsonl):
// {"Name":"Alice","Age":30}
// {"Name":"Bob","Age":25}

using var stream = File.OpenRead("data.jsonl");
var extractor = new JsonLineExtractor<Person>(stream, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine($"{person.Name}, age {person.Age}");
}
```

## JsonLineLoader

`JsonLineLoader<TRecord>` inherits `LoaderBase<TRecord, JsonReport>`.

Writes each item as a single line of JSON to the output stream, with each line separated by a newline character.

### Constructors

```csharp
// Basic:
public JsonLineLoader
(
    Stream stream,
    ILogger<JsonLineLoader<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonLineLoader
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonLineLoader<TRecord>> logger
);
```

### Example

```csharp
using var stream = File.Create("output.jsonl");
var loader = new JsonLineLoader<Person>(stream, logger);

await loader.LoadAsync(personStream, cancellationToken);

// Output:
// {"Name":"Alice","Age":30}
// {"Name":"Bob","Age":25}
```

## JsonSingleStreamExtractor

`JsonSingleStreamExtractor<TRecord>` inherits `ExtractorBase<TRecord, JsonReport>`.

Reads a JSON array from a stream using `JsonSerializer.DeserializeAsyncEnumerable<T>` for true streaming deserialization. The entire array is never buffered in memory.

### Constructors

```csharp
// Basic:
public JsonSingleStreamExtractor
(
    Stream stream,
    ILogger<JsonSingleStreamExtractor<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonSingleStreamExtractor
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonSingleStreamExtractor<TRecord>> logger
);
```

### Example

```csharp
// Input (data.json):
// [
//   {"Name":"Alice","Age":30},
//   {"Name":"Bob","Age":25}
// ]

using var stream = File.OpenRead("data.json");
var extractor = new JsonSingleStreamExtractor<Person>(stream, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine($"{person.Name}, age {person.Age}");
}
```

## JsonSingleStreamLoader

`JsonSingleStreamLoader<TRecord>` inherits `LoaderBase<TRecord, JsonReport>`.

Writes a JSON array to a stream using `Utf8JsonWriter` for efficient UTF-8 output. Each item is serialized into the array using `JsonSerializer.Serialize`.

### Constructors

```csharp
// Basic:
public JsonSingleStreamLoader
(
    Stream stream,
    ILogger<JsonSingleStreamLoader<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonSingleStreamLoader
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonSingleStreamLoader<TRecord>> logger
);
```

### Example

```csharp
using var stream = File.Create("output.json");
var loader = new JsonSingleStreamLoader<Person>(stream, logger);

await loader.LoadAsync(personStream, cancellationToken);

// Output:
// [{"Name":"Alice","Age":30},{"Name":"Bob","Age":25}]
```

## JsonMultiStreamExtractor

`JsonMultiStreamExtractor<TRecord>` inherits `ExtractorBase<TRecord, JsonReport>`.

Iterates over an `IEnumerable<Stream>`, deserializing one JSON object from each stream using `JsonSerializer.DeserializeAsync<T>`. Each stream is disposed after reading. Streams that deserialize to `null` are skipped.

### Constructors

```csharp
// Basic:
public JsonMultiStreamExtractor
(
    IEnumerable<Stream> streams,
    ILogger<JsonMultiStreamExtractor<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonMultiStreamExtractor
(
    IEnumerable<Stream> streams,
    JsonSerializerOptions options,
    ILogger<JsonMultiStreamExtractor<TRecord>> logger
);
```

### Example

```csharp
var streams = Directory.GetFiles("data/", "*.json")
    .Select(path => (Stream)File.OpenRead(path));

var extractor = new JsonMultiStreamExtractor<Person>(streams, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine(person.Name);
}
```

## JsonMultiStreamLoader

`JsonMultiStreamLoader<TRecord>` inherits `LoaderBase<TRecord, JsonReport>`.

For each item in the input sequence, calls a factory function to obtain a `Stream`, serializes the item as a single JSON object using `JsonSerializer.SerializeAsync`, flushes, and disposes the stream. The factory receives the item, so stream creation can depend on item properties.

The factory must not return `null`; doing so throws `InvalidOperationException`.

### Constructors

```csharp
// Basic:
public JsonMultiStreamLoader
(
    Func<TRecord, Stream> streamFactory,
    ILogger<JsonMultiStreamLoader<TRecord>> logger
);

// With custom JsonSerializerOptions:
public JsonMultiStreamLoader
(
    Func<TRecord, Stream> streamFactory,
    JsonSerializerOptions options,
    ILogger<JsonMultiStreamLoader<TRecord>> logger
);
```

### Example

```csharp
var loader = new JsonMultiStreamLoader<Person>
(
    person => File.Create($"output/{person.Id}.json"),
    logger
);

await loader.LoadAsync(personStream, cancellationToken);
// Creates one JSON file per person: output/1.json, output/2.json, etc.
```

## JsonReport

Extends `Report` with JSON-specific progress data.

| Property | Type | Description |
|---|---|---|
| `CurrentCount` | `int` | Number of records processed (inherited). |
| `CurrentSkippedItemCount` | `int` | Number of records skipped. |

## JsonSerializerOptions customization

All six classes accept an optional `JsonSerializerOptions` for controlling serialization behavior. Common customizations:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

var extractor = new JsonLineExtractor<Person>(stream, options, logger);
```

When no options are provided:
- `JsonLineExtractor` and `JsonLineLoader` use `null` (System.Text.Json defaults).
- `JsonSingleStreamExtractor` creates `new JsonSerializerOptions()` for `DeserializeAsyncEnumerable`.
- `JsonSingleStreamLoader` uses `null` (defaults).
- Multi-stream classes use `null` (defaults).

## Format comparison

| Feature | JSONL | JSON Array | Multi-Stream |
|---|---|---|---|
| File count | 1 | 1 | N |
| Streaming read | Line-by-line | `DeserializeAsyncEnumerable` | One per stream |
| Memory usage | One object at a time | One object at a time | One object at a time |
| Random access | Line number | No | File-based |
| Best for | Log files, data pipelines | API responses, datasets | File-per-record storage |

## Common patterns

### SkipItemCount and MaximumItemCount

All six classes support `SkipItemCount` and `MaximumItemCount` from the base class:

```csharp
var extractor = new JsonLineExtractor<Person>(stream, logger);
extractor.SkipItemCount = 10;      // Skip first 10 records
extractor.MaximumItemCount = 100;   // Extract at most 100 records
```

### JSONL round-trip

```csharp
// Extract from one JSONL file, transform, and load to another:
using var inputStream = File.OpenRead("input.jsonl");
using var outputStream = File.Create("output.jsonl");

var extractor = new JsonLineExtractor<InputRecord>(inputStream, logger);
var loader = new JsonLineLoader<OutputRecord>(outputStream, logger);

// Use with a transformer or manual pipeline:
await foreach (var record in extractor.ExtractAsync(cancellationToken))
{
    var transformed = Transform(record);
    // ... load via loader
}
```

### JSON array to JSONL conversion

```csharp
using var jsonArrayStream = File.OpenRead("data.json");
using var jsonlStream = File.Create("data.jsonl");

var extractor = new JsonSingleStreamExtractor<Record>(jsonArrayStream, logger);
var loader = new JsonLineLoader<Record>(jsonlStream, logger);
```

### Null handling

All extractors skip `null` deserialization results with a log message. This handles:
- JSONL lines that deserialize to `null`
- JSON array elements that are `null`
- Multi-stream files that contain `null` JSON
