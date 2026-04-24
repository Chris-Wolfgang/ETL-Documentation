# Wolfgang.Etl Framework

The Wolfgang.Etl framework is a set of .NET libraries for building ETL (Extract, Transform, Load) pipelines. Each pipeline stage is built on abstract base classes that provide cancellation, progress reporting, pagination, and async streaming out of the box. Stages are wired together using `IAsyncEnumerable<T>`, creating a lazy, pull-based pipeline where items flow one at a time without buffering the entire dataset in memory.

## How the Pipeline Works

The pipeline uses a **pull model**. The loader drives execution:

```csharp
await loader.LoadAsync
(
    transformer.TransformAsync
    (
        extractor.ExtractAsync(cancellationToken),
        cancellationToken
    ),
    cancellationToken
);
```

1. The loader calls `LoadAsync`, which starts enumerating items
2. Each pull triggers the transformer to yield the next transformed item
3. Each transformer pull triggers the extractor to read and yield the next source item

No intermediate collections. Memory usage stays constant regardless of dataset size.

## Packages

The framework ships as several focused NuGet packages. See [Installation](getting-started/installation.md) for the full list and install commands, or browse [Libraries](libraries/fixedwidth.md) for what each format-specific package does.

## Built-in Capabilities

Every extractor, transformer, and loader built on the base classes gets these features for free:

| Feature | Description |
|---------|-------------|
| **Cancellation** | Pass a `CancellationToken` to any async method |
| **Progress reporting** | Pass an `IProgress<TProgress>` to receive periodic callbacks |
| **Skip items** | Set `SkipItemCount` to skip the first N items |
| **Limit items** | Set `MaximumItemCount` to stop after N items |
| **Async streaming** | Items flow one at a time via `IAsyncEnumerable<T>` |
| **Interchangeability** | Any extractor can replace any other extractor of the same `TSource` |

## What You Need to Build an ETL

| Component | Count | Description |
|-----------|-------|-------------|
| Source type | 1 | A class/record representing the raw data |
| Destination type | 1 | A class/record representing the target data |
| Extractor | 1 | Reads source data and yields `IAsyncEnumerable<TSource>` |
| Transformer | 1-N | Converts `TSource` to `TDestination` |
| Loader | 1 | Consumes `IAsyncEnumerable<TDestination>` and writes to the target |

!!! tip "Every ETL has a transformer stage"
    When no transformation is needed — for example, reading a fixed width file and writing it to a CSV file — use `PassThroughTransformer<T>` from `Wolfgang.Etl.Transformers`. It pulls each item from the extractor and yields it unchanged, preserving the three-stage shape.

    Connecting the loader directly to the extractor is an antipattern: it breaks the three-stage shape the rest of the framework is built around, and adding a transformer later (for logging, validation, enrichment, rate limiting, etc.) requires restructuring the pipeline.

!!! note "PassThroughTransformer<T> is a planned class"
    Tracked by [Chris-Wolfgang/ETL-Transformers#1](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/1). Until it ships, write a small pass-through `TransformerBase<T, T, TProgress>` subclass whose `TransformWorkerAsync` iterates the input and yields each item after calling `IncrementCurrentItemCount()`.

## Quick Links

- **[Installation](getting-started/installation.md)** — get started with NuGet packages
- **[Your First Extractor](getting-started/your-first-extractor.md)** — build an extractor from scratch
- **[Your First Loader](getting-started/your-first-loader.md)** — build a loader from scratch
- **[Architecture](architecture.md)** — base classes, interfaces, and design patterns
- **[Libraries](libraries/fixedwidth.md)** — FixedWidth, DbClient, XML, JSON reference docs
- **[Guides](guides/pipeline-composition.md)** — pipeline composition, error handling, progress, performance
