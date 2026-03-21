# Architecture

The Wolfgang.Etl framework is organized into layers: **Abstractions** define the contracts, **ETL Libraries** implement them for specific formats, and **your application** wires them together.

```
┌─────────────────────────────────────────────────────────────────┐
│  Your Application                                               │
│  ┌───────────┐    ┌───────────────┐    ┌───────────────────┐   │
│  │ Extractor  │───>│  Transformer  │───>│      Loader       │   │
│  │ (FixedWidth│    │  (your code)  │    │    (DbClient)     │   │
│  │  JSON, Xml)│    │               │    │  FixedWidth, JSON │   │
│  └───────────┘    └───────────────┘    └───────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  ETL Libraries                                                  │
│  Wolfgang.Etl.FixedWidth  │  Wolfgang.Etl.DbClient              │
│  Wolfgang.Etl.Json        │  Wolfgang.Etl.Xml                   │
├─────────────────────────────────────────────────────────────────┤
│  Wolfgang.Etl.Abstractions                                      │
│  ExtractorBase  │  TransformerBase  │  LoaderBase  │  Report    │
│  IExtractAsync  │  ITransformAsync  │  ILoadAsync  │  IProgress │
├─────────────────────────────────────────────────────────────────┤
│  Wolfgang.Etl.TestKit / TestKit.Xunit                           │
│  TestExtractor  │  TestTransformer  │  TestLoader               │
│  ExtractorBaseContractTests  │  LoaderBaseContractTests          │
└─────────────────────────────────────────────────────────────────┘
```

## Base Classes

### ExtractorBase&lt;TSource, TProgress&gt;

The base class for all extractors. Reads data from a source and yields items as `IAsyncEnumerable<TSource>`.

- `TSource` must be `notnull`
- Implement `ExtractWorkerAsync(CancellationToken)` — the async iterator that yields items
- Implement `CreateProgressReport()` — returns a `TProgress` snapshot
- Call `IncrementCurrentItemCount()` exactly once per yielded item
- Implement `SkipItemCount` and `MaximumItemCount` logic in your worker method (the base class does not enforce them)

### TransformerBase&lt;TSource, TDestination, TProgress&gt;

Converts items from one type to another. Receives `IAsyncEnumerable<TSource>` and yields `IAsyncEnumerable<TDestination>`.

- Implement `TransformWorkerAsync(IAsyncEnumerable<TSource>, CancellationToken)`
- Same counting and skip/max rules as extractors

### LoaderBase&lt;TDestination, TProgress&gt;

Consumes `IAsyncEnumerable<TDestination>` and writes items to a destination.

- Implement `LoadWorkerAsync(IAsyncEnumerable<TDestination>, CancellationToken)`
- Same counting and skip/max rules as extractors

## Interfaces

Each base class implements a corresponding interface:

| Interface | Methods |
|-----------|---------|
| `IExtractAsync<TSource, TProgress>` | `ExtractAsync()`, `ExtractAsync(CancellationToken)`, `ExtractAsync(IProgress<TProgress>)`, `ExtractAsync(IProgress<TProgress>, CancellationToken)` |
| `ITransformAsync<TSource, TDestination, TProgress>` | Same four overloads for `TransformAsync` |
| `ILoadAsync<TDestination, TProgress>` | Same four overloads for `LoadAsync` |

The interfaces enable mocking and substitution in tests without depending on the base class implementation.

## Report and TProgress

The `Report` base class provides `CurrentItemCount`. Libraries extend it with domain-specific properties:

| Report Type | Library | Extra Properties |
|-------------|---------|-----------------|
| `Report` | Abstractions | `CurrentItemCount` |
| `FixedWidthReport` | FixedWidth | `CurrentSkippedItemCount`, `CurrentLineNumber` |
| `DbReport` | DbClient | `CurrentSkippedItemCount`, `CommandText`, `ElapsedMilliseconds` |
| `JsonReport` | Json | `CurrentSkippedItemCount` |
| `XmlReport` | Xml | `CurrentSkippedItemCount` |

Progress reports are created by `CreateProgressReport()` and delivered via `IProgress<TProgress>` on a timer thread.

## IProgressTimer Pattern

The base classes use `IProgressTimer` to abstract the system timer for testability:

- **Production**: `SystemProgressTimer` wraps `System.Threading.Timer`
- **Testing**: `ManualProgressTimer` (from TestKit) lets you fire the timer on demand

Each ETL component follows a consistent injection pattern:

1. `private readonly IProgressTimer? _progressTimer` field
2. `private bool _progressTimerWired` guard flag
3. `internal` constructor that accepts `IProgressTimer`
4. `CreateProgressTimer` override that wires the injected timer

## The notnull Constraint

All `TRecord`/`TSource`/`TDestination` type parameters require `notnull`. This is enforced by Abstractions 0.10.2. Use classes or non-nullable value types — nullable reference types and `Nullable<T>` are not valid record types.

## TestKit Architecture

### Test Doubles

| Class | Purpose |
|-------|---------|
| `TestExtractor<T>` | Yields items from a list — use as a stand-in extractor in integration tests |
| `TestLoader<T>` | Captures loaded items into a list — verify what was written |
| `TestTransformer<T>` | Passes items through unchanged — verify pipeline wiring |

### Contract Test Base Classes

| Class | Tests |
|-------|-------|
| `ExtractorBaseContractTests<TSut, TItem, TProgress>` | 20+ tests verifying ExtractorBase contract |
| `LoaderBaseContractTests<TSut, TItem, TProgress>` | 20+ tests verifying LoaderBase contract |

Implement three factory methods (`CreateSut`, `CreateExpectedItems`/`CreateSourceItems`, `CreateSutWithTimer`) and the base class provides comprehensive contract verification: all four async overloads, cancellation, skip/max, progress callbacks, empty source handling, and counting correctness.

## Multi-TFM Strategy

| Project Type | Target Frameworks |
|-------------|-------------------|
| Source libraries | `net462;net481;netstandard2.0;net8.0;net10.0` |
| Test projects | `net462;net47;net471;net472;net48;net481;netcoreapp3.1;net5.0;net6.0;net7.0;net8.0;net9.0;net10.0` |

Source libraries target `netstandard2.0` for maximum compatibility and `net8.0`/`net10.0` for span-based optimizations. Test projects target the full matrix to verify behavior across all supported runtimes.
