# Wolfgang.Etl.Json

JSON extractor and loader for JSONL, JSON arrays, and per-file multi-stream.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.Json` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.Json) |
| **Dependencies** | Wolfgang.Etl.Abstractions, System.Text.Json, Microsoft.Extensions.Logging.Abstractions |
| **Target Frameworks** | net462, net481, netstandard2.0, net8.0, net10.0 |
| **Source** | [ETL-Json](https://github.com/Chris-Wolfgang/ETL-Json) |

## What it does

Provides three extractor/loader pairs, each tuned to a different JSON file layout:

| Format | Extractor | Loader | Description |
|---|---|---|---|
| **JSONL / NDJSON** | `JsonLineExtractor<T>` | `JsonLineLoader<T>` | One JSON object per line |
| **JSON Array** | `JsonSingleStreamExtractor<T>` | `JsonSingleStreamLoader<T>` | A single JSON array `[{...},{...}]` |
| **Multi-Stream** | `JsonMultiStreamExtractor<T>` | `JsonMultiStreamLoader<T>` | One JSON object per stream or file |

All six classes stream records without buffering the entire input, and accept an optional serializer-options object for customizing property naming, indentation, null handling, custom converters, and so on.

## When to use it

- **JSONL / NDJSON** — log files, append-only data pipelines, per-line processing
- **JSON Array** — API responses, exported datasets with a single-document shape
- **Multi-Stream** — one-file-per-record storage (e.g. dumping each order to its own `.json` file), or reading a directory of individual JSON files
- When you want **serialization customization** — property naming conventions, indentation, null handling, custom converters

## When to use something else

- For XML, use `Wolfgang.Etl.Xml`
- For arbitrary JSON that doesn't map cleanly to a .NET type, process the raw JSON with your own parser and build a custom extractor

## Quick example

```csharp
// Convert JSONL -> single JSON array
using var inputStream = File.OpenRead("data.jsonl");
var extractor = new JsonLineExtractor<Person>(inputStream, logger);

using var outputStream = File.Create("output.json");
var loader = new JsonSingleStreamLoader<Person>(outputStream, logger);

await loader.LoadAsync(extractor.ExtractAsync(cancellationToken), cancellationToken);
```

## More detail

- **API reference** — see the [ETL-Json README](https://github.com/Chris-Wolfgang/ETL-Json#readme) and XML doc comments on the public types
- **Examples** — see the [`examples/` folder](https://github.com/Chris-Wolfgang/ETL-Json/tree/main/examples) in the source repo
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-Json/releases)
