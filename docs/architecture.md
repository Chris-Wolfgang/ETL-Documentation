# Architecture

The Wolfgang.Etl framework is organized into layers: **Abstractions** define the contracts, **ETL Libraries** implement them for specific formats, and **your application** wires them together.

![Wolfgang.Etl framework architecture](assets/architecture-light.svg#only-light)
![Wolfgang.Etl framework architecture](assets/architecture-dark.svg#only-dark)

## Interfaces

Interfaces are the contract every component is built on. Each stage has **four** interfaces arranged in a diamond inheritance pattern — not a single interface with four overloads. Implementing the base interface gives you the bare minimum; each subsequent interface layers on an additional capability:

![IExtractAsync interface diamond](assets/interface-diamond-light.svg#only-light)
![IExtractAsync interface diamond](assets/interface-diamond-dark.svg#only-dark)

| Interface | Adds |
|-----------|------|
| `IExtractAsync<TSource>` | `ExtractAsync()` returning `IAsyncEnumerable<TSource>` — the bare minimum |
| `IExtractWithCancellationAsync<TSource>` | `CancellationToken` overload |
| `IExtractWithProgressAsync<TSource, TProgress>` | `IProgress<TProgress>` overload |
| `IExtractWithProgressAndCancellationAsync<TSource, TProgress>` | Both combined |

The cancellation and progress interfaces both inherit from `IExtractAsync<TSource>`. The combined interface inherits from both, forming the diamond. The same four-interface structure exists for `ILoad*Async<TDestination[, TProgress]>` and `ITransform*Async<TSource, TDestination[, TProgress]>`.

Implement only the interface your component actually needs. An extractor that does not meaningfully support cancellation should not implement a cancellation interface — declare intent through what you implement, not through what you no-op.

The interfaces enable mocking and substitution in tests without depending on the base class implementation. `ExtractorBase`, `TransformerBase`, and `LoaderBase` (described in the next section) each implement all four interfaces of their stage, so most consumers never need to think about the diamond — but if you need full control (e.g. wrapping a third-party reader with an incompatible lifecycle), implementing an interface directly is supported. See [Your First Extractor](getting-started/your-first-extractor.md) and [Your First Loader](getting-started/your-first-loader.md) for the full "Interfaces vs. the Base Class" discussion.

## Base Classes

The interfaces above are the contract; the base classes are the easy way to implement that contract. Inheriting from `ExtractorBase`, `TransformerBase`, or `LoaderBase` gives you all four interface implementations plus orchestration (cancellation wiring, progress reporting, skip/max enforcement, the timer-injection pattern) for free, leaving you to implement just one worker method.

### ExtractorBase&lt;TSource, TProgress&gt;

The base class for all extractors. Reads data from a source and yields items as `IAsyncEnumerable<TSource>`.

- `TSource` must be `notnull`
- Implement `ExtractWorkerAsync(CancellationToken)` — the async iterator that yields items
- Implement `CreateProgressReport()` — returns a `TProgress` snapshot
- Call `IncrementCurrentItemCount()` exactly once per yielded item
- Implement `SkipItemCount` and `MaximumItemCount` logic in your worker method (the base class does not enforce them)

Classes derived from `ExtractorBase` and `LoaderBase` are generally reusable across applications. Reading from (or writing to) a database, CSV, JSON, or a company's ERP / CRM / service bus is mostly the same from one ETL to the next — so making your extractor or loader as generic as reasonable for your situation is recommended.

### TransformerBase&lt;TSource, TDestination, TProgress&gt;

Converts items from one type to another. Receives `IAsyncEnumerable<TSource>` and yields `IAsyncEnumerable<TDestination>`.

- Implement `TransformWorkerAsync(IAsyncEnumerable<TSource>, CancellationToken)`
- Same counting and skip/max rules as extractors

In practice, transformers tend to fall at one of two extremes: very simple (e.g. `PassThroughTransformer<T>` or `SelectTransformer<TSource, TDestination>`) or very complex business-logic-heavy mappings — there is rarely much in the middle. Because a transformer takes `IAsyncEnumerable<TSource>` in and yields `IAsyncEnumerable<TDestination>` out, you can fully unit-test it in isolation, without connecting it to a real extractor or loader. A `TestExtractor<T>` or a hand-rolled `IAsyncEnumerable<T>` source is enough.

!!! note "PassThroughTransformer<T> and SelectTransformer<TSource, TDestination> are planned classes"
    Both will live in `Wolfgang.Etl.Transformers`. `PassThroughTransformer` is tracked by [Chris-Wolfgang/ETL-Transformers#1](https://github.com/Chris-Wolfgang/ETL-Transformers/issues/1); `SelectTransformer` is in [Chris-Wolfgang/ETL-Transformers#8](https://github.com/Chris-Wolfgang/ETL-Transformers/pull/8) (open PR).

### LoaderBase&lt;TDestination, TProgress&gt;

Consumes `IAsyncEnumerable<TDestination>` and writes items to a destination.

- Implement `LoadWorkerAsync(IAsyncEnumerable<TDestination>, CancellationToken)`
- Same counting and skip/max rules as extractors

Classes derived from `ExtractorBase` and `LoaderBase` are generally reusable across applications. Reading from (or writing to) a database, CSV, JSON, or a company's ERP / CRM / service bus is mostly the same from one ETL to the next — so making your extractor or loader as generic as reasonable for your situation is recommended.

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

All `TRecord`/`TSource`/`TDestination` type parameters require `notnull`. Use classes or non-nullable value types — nullable reference types and `Nullable<T>` are not valid record types.

## TestKit Architecture

`Wolfgang.Etl.TestKit` and `Wolfgang.Etl.TestKit.Xunit` provide test doubles and contract test base classes for verifying components built on `Wolfgang.Etl.Abstractions`. The summary below is for architectural orientation — see the [TestKit catalog page](libraries/testkit.md) for the package overview and the [TestKit reference](reference/testkit.md) for the full API.

### Test Doubles

| Class | Purpose |
|-------|---------|
| [`TestExtractor<T>`](reference/testkit.md) | Yields items from a list — use as a stand-in extractor in integration tests |
| [`TestLoader<T>`](reference/testkit.md) | Captures loaded items into a list — verify what was written |
| [`TestTransformer<T>`](reference/testkit.md) | Passes items through unchanged — verify pipeline wiring |

### Contract Test Base Classes

| Class | Tests |
|-------|-------|
| [`ExtractorBaseContractTests<TSut, TItem, TProgress>`](reference/testkit.md) | 20+ tests verifying ExtractorBase contract |
| [`LoaderBaseContractTests<TSut, TItem, TProgress>`](reference/testkit.md) | 20+ tests verifying LoaderBase contract |
| [`TransformerBaseContractTests<TSut, TItem, TProgress>`](reference/testkit.md) | 20+ tests verifying TransformerBase contract |

Implement three factory methods (`CreateSut`, `CreateExpectedItems`/`CreateSourceItems`, `CreateSutWithTimer`) and the base class provides comprehensive contract verification: all four async overloads, cancellation, skip/max, progress callbacks, empty source handling, and counting correctness.

## Multi-TFM Strategy

| Project Type | Target Frameworks |
|-------------|-------------------|
| Source libraries | `net462;net481;netstandard2.0;net8.0;net10.0` |
| Test projects | `net462;net47;net471;net472;net48;net481;netcoreapp3.1;net5.0;net6.0;net7.0;net8.0;net9.0;net10.0` |

Source libraries target `netstandard2.0` for maximum compatibility and `net8.0`/`net10.0` for span-based optimizations. Test projects target the full matrix to verify behavior across all supported runtimes.
