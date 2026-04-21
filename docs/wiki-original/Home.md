# Wolfgang.Etl Framework

This wiki is the comprehensive guide for building ETL pipelines using the Wolfgang.Etl packages. Whether you are creating a new extractor, loader, or transformer from scratch, or wiring them together into a complete pipeline, you will find everything you need here.

## What is this framework?

The ETL design pattern involves three stages:
- **Extract** -- Retrieve data from a source (file, database, API, message queue)
- **Transform** -- Convert data from the source format to the destination format
- **Load** -- Write data to the destination

This framework provides abstract base classes and interfaces that let you build each stage independently. The stages are wired together using `IAsyncEnumerable<T>`, creating a lazy, pull-based pipeline where items flow one at a time without buffering the entire dataset in memory.

## How the pipeline works

The pipeline uses a **pull model**. The loader drives execution:

```
await loader.LoadAsync(transformer.TransformAsync(extractor.ExtractAsync()));
```

1. The loader calls `LoadAsync`, which starts enumerating items
2. Each pull triggers the transformer to yield the next transformed item
3. Each transformer pull triggers the extractor to read and yield the next source item

No intermediate collections. Memory usage stays constant regardless of dataset size.

## What you need to build an ETL

| Component | Class Count | Description |
|-----------|-------------|-------------|
| Source type | 1 | A class/record representing the raw data |
| Destination type | 1 | A class/record representing the target data |
| Extractor | 1 | Reads source data and yields `IAsyncEnumerable<TSource>` |
| Transformer | 1-N | Converts `TSource` to `TDestination` |
| Loader | 1 | Consumes `IAsyncEnumerable<TDestination>` and writes to the target |

> Every ETL has a transformer stage. If you are moving data without converting it, use `NoOpTransformer<T>` (from `Wolfgang.Etl.Transformers`) -- a pass-through transformer that pulls each item from the extractor and yields it unchanged. Skipping the transformer entirely is an antipattern because it breaks the three-stage shape of the pipeline and makes later additions (logging, validation, enrichment) harder.
>
> **Note:** `NoOpTransformer<T>` is a planned class tracked by [Chris-Wolfgang/ETL-Transformers#1](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/1). Until it ships, users can write a small pass-through `TransformerBase<T, T, TProgress>` subclass.

## Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `Wolfgang.Etl.Abstractions` | 0.10.2 | Base classes and interfaces for building ETL components |
| `Wolfgang.Etl.TestKit` | 0.5.0 | Test doubles (`TestExtractor`, `TestLoader`, `TestTransformer`) |
| `Wolfgang.Etl.TestKit.Xunit` | 0.5.0 | Contract test base classes for xUnit |
| `Wolfgang.Etl.Json` | 0.1.0 | JSON, JSONL, and multi-stream JSON extractors and loaders |

## Guides

Start here if you are building a new ETL library or component:

- **[Building an Extractor](Building-an-Extractor)** -- step-by-step guide using `JsonLineExtractor` as a real example
- **[Building a Loader](Building-a-Loader)** -- step-by-step guide using `JsonLineLoader` as a real example
- **[Building a Complete ETL](Building-a-Complete-ETL)** -- wiring extractor, transformer, and loader into a pipeline
- **[TestKit](TestKit)** -- contract test base classes, test doubles, and the timer injection pattern

## API Reference

Detailed interface and base class documentation:

- **[Extractor](Extractor)** -- `IExtractAsync`, `ExtractorBase<TSource, TProgress>`
- **[Transformer](Transformer)** -- `ITransformAsync`, `TransformerBase<TSource, TDestination, TProgress>`
- **[Loader](Loader)** -- `ILoadAsync`, `LoaderBase<TDestination, TProgress>`

## Built-in capabilities

Every extractor, transformer, and loader built on the base classes gets these features for free:

| Feature | Description |
|---------|-------------|
| **Cancellation** | Pass a `CancellationToken` to any async method |
| **Progress reporting** | Pass an `IProgress<TProgress>` to receive periodic callbacks |
| **Skip items** | Set `SkipItemCount` to skip the first N items |
| **Limit items** | Set `MaximumItemCount` to stop after N items |
| **Async streaming** | Items flow one at a time via `IAsyncEnumerable<T>` |
| **Interchangeability** | Any extractor can replace any other extractor of the same `TSource` |
