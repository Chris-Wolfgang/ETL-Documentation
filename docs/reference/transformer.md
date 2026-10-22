# Transformer

API reference for the transformer interfaces and the `TransformerBase<TSource, TDestination, TProgress>` abstract base class in `Wolfgang.Etl.Abstractions`.

For a step-by-step guide to building a concrete transformer, see [Your First Transformer](../getting-started/your-first-transformer.md). For wiring it into a complete pipeline, see [Pipeline Composition](../guides/pipeline-composition.md). For the underlying architecture, see [Architecture](../architecture.md).

The transformer is the second step in the ETL process, converting data from the source format, `TSource`, to the destination format, `TDestination`.


## Requirements

1. A transformer is usually **application-specific**, not generic. The mapping from `ErpCustomer` to `CrmAccount` is inherently particular to your ETL — which fields map where, how values are translated between systems, what defaults apply when a field is missing. Trying to force a single generic transformer to cover these variations usually produces an abstraction that fits no real use case well.

    The pattern for reuse is not one giant generic transformer but many small focused ones that compose through shared intermediate types. For example, address validation is genuinely generic:

    ```
    ErpAddress --(ErpAddressToAddress)--> Address --(AddressValidator)--> Address --(AddressToErpAddress)--> ErpAddress
    CrmAddress --(CrmAddressToAddress)--> Address --(AddressValidator)--> Address --(AddressToCrmAddress)--> CrmAddress
    ```

    The `AddressValidator` in the middle is reusable across every ETL that works with addresses. The shape-conversion transformers on either side are application-specific and small enough to be easy to write and test.

    Use `Then(...)` / [`ChainTransformer<TSource, TIntermediate, TDestination>`](https://chris-wolfgang.github.io/ETL-Transformers/api/Wolfgang.Etl.Transformers.ChainTransformer-3.html) from `Wolfgang.Etl.Transformers` to compose a chain like this into a single transformer you can hand to the loader — see [Pipeline Composition § Chaining Transformers](../guides/pipeline-composition.md#chaining-transformers).

    !!! note "Wolfgang.Etl.Transformers"
        Generic transformers that are broadly reusable (validators, format converters, pass-through) live in the published `Wolfgang.Etl.Transformers` package: pass-through, select / where / distinct / chunk / take / skip, buffering, throttling and chaining — see [Chris-Wolfgang/ETL-Transformers](https://github.com/Chris-Wolfgang/ETL-Transformers).
1. Each `Transformer<TSource, TDestination>` should be interchangeable for any other `Transformer<TSource, TDestination>`, where `TSource` and `TDestination` are of the same type. This allows your application to swap transformers at compile time or runtime, making your ETL more flexible.
1. A transformer is responsible for handling any exceptions that may occur during the transformation process, including but not limited to invalid values and missing values.
1. The data to transform is received as an asynchronous stream of type `TSource`, transformed, and returned as an asynchronous stream of type `TDestination`, allowing for efficient processing by any additional transformer(s) and the loader.
1. The transformer is decoupled from the source and destination data, meaning it does not need to know where the data is coming from or where it is going. It only needs to know how to transform the data from one type to another. This means the transformer can be easily tested using unit tests and does not need any integration tests to verify its functionality.


## Interfaces

The four transformer interfaces form a diamond — see [Architecture § Interfaces](../architecture.md#interfaces) for the shape shared across all three stages.

| Interface | Description |
|-----------|-------------|
| [ITransformAsync](#itransformasync) | A method to transform data asynchronously from source to destination. |
| [ITransformWithCancellationAsync](#itransformwithcancellationasync) | A method to transform data asynchronously with the ability to cancel. |
| [ITransformWithProgressAsync](#itransformwithprogressasync) | A method to transform data asynchronously while reporting progress. |
| [ITransformWithProgressAndCancellationAsync](#itransformwithprogressandcancellationasync) | A method to transform data asynchronously with both progress reporting and cancellation. |


### ITransformAsync

#### Type Parameters

`TSource` — the type of elements being passed into the transformer. This represents your source data.

`TDestination` — the type of elements being returned. This represents your destination or target data.

#### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>)` | Transforms data from source format to destination format asynchronously |

#### TransformAsync\<TSource, TDestination\>

Transforms data from the source format to the destination format by enumerating it asynchronously.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

**Introduced in:** 0.4.0


### ITransformWithCancellationAsync

#### Type Parameters

`TSource` — the type of elements being passed in. This represents your source data.

`TDestination` — the type of elements being returned. This represents your destination or target data.

#### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, CancellationToken cancellationToken)` | Transforms data asynchronously with the ability to cancel |

#### TransformAsync\<TSource, TDestination\>

Transforms data from the source format to the destination format by enumerating it asynchronously.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

**Introduced in:** 0.4.0


### ITransformWithProgressAsync\<TSource, TDestination, TProgress\>

#### Type Parameters

`TSource` — the type of elements being passed in. This represents your source data.

`TDestination` — the type of elements being returned. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, IProgress<TProgress> progress)` | Transforms data asynchronously reporting progress |

#### TransformAsync\<TSource, TDestination\>

Transforms data from the source format to the destination format by enumerating it asynchronously.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the transformer.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

**Introduced in:** 0.4.0


### ITransformWithProgressAndCancellationAsync\<TSource, TDestination, TProgress\>

#### Type Parameters

`TSource` — the type of elements being passed in. This represents your source data.

`TDestination` — the type of elements being returned. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource>, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Transforms data asynchronously reporting progress with the ability to cancel |

#### TransformAsync\<TSource, TDestination, TProgress\>

Transforms data from the source format to the destination format by enumerating it asynchronously.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the transformer.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

**Introduced in:** 0.4.0


## TransformerBase\<TSource, TDestination, TProgress\>

### Definition

Provides an abstract base class that implements [`ITransformAsync`](#itransformasync), [`ITransformWithCancellationAsync`](#itransformwithcancellationasync), [`ITransformWithProgressAsync`](#itransformwithprogressasync), and [`ITransformWithProgressAndCancellationAsync`](#itransformwithprogressandcancellationasync) with basic functionality, allowing for quickly building user-defined transformers. This abstract class only requires that you implement two methods: [`CreateProgressReport`](#createprogressreport) and [`TransformWorkerAsync`](#transformworkerasync).

### Type Parameters

`TSource` — the type of elements being passed in. This represents your source data.

`TDestination` — the type of elements being returned. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

### Constructors

| Constructor | Description |
|-------------|-------------|
| `TransformerBase(TransformerOptions? options = null)` | *(protected)* Initializes the stage from an [`TransformerOptions`](#transformeroptions) record. `null` or omitted applies the defaults; out-of-range values throw `ArgumentOutOfRangeException` here. |
| `TransformerBase()` | *(protected, hidden from IntelliSense)* Kept for binary compatibility with assemblies compiled before the options constructor existed; same as passing `null`. |

### Properties

| Property | Description |
|----------|-------------|
| `CurrentItemCount` | *(get)* The number of items transformed so far in the current run. |
| `CurrentSkippedItemCount` | *(get)* The number of items skipped so far in the current run (`SkipItemCount`, or items a worker chose to skip). |
| `CurrentErrorItemCount` | *(get)* The number of items that failed and were discarded by the error policy. |
| `MaximumItemCount` | The maximum number of items to process before stopping. Default `int.MaxValue`. |
| `SkipItemCount` | The number of items to skip before processing. Default `0`. |
| `ReportingInterval` | The number of milliseconds between progress reports. Default `1000`. |
| `ErrorPolicy` | *(init)* `Func<ItemErrorContext, ItemErrorAction>` deciding what happens when an item fails: `ItemErrorAction.Abort` (default) or `ItemErrorAction.Skip`. Ready-made policies are in `Wolfgang.Etl.ErrorPolicies`. |
| `WorkerResilience` | *(init)* A wrapper around each worker invocation, for example a retry policy. A retry re-runs the whole worker. |
| `StartedAt` | *(protected get)* The UTC time the first item was processed, or `null` before that. |
| `Elapsed` | *(protected get)* Wall-clock time since the first item was processed; use it when building a `Report`. |

!!! note "Configure the limits through the options record"
    `MaximumItemCount`, `SkipItemCount` and `ReportingInterval` still have public setters, but the setters are `[Obsolete]` and will be removed (ADR-0009): pass the values in the [`TransformerOptions`](#transformeroptions) record (or your own options record derived from it) when the stage is constructed. A value assigned after a run has started is not honoured.


#### CurrentItemCount

Read-only. The current number of items that have been transformed. This property is updated as items are transformed and can be used to report progress.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the current number of items that have been transformed.

##### Default Value

`0`.

**Introduced in:** 0.4.0


#### MaximumItemCount

The maximum number of items to be transformed before exiting and reporting complete. This property can be used to limit the number of items transformed, which is useful for testing or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the maximum number of items to transform.

##### Default Value

[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) — by default there is no limit on the number of items transformed.

> If you are trying to transform a large number of items, you are limited to the maximum value of a 32-bit integer, which is 2,147,483,647. If you need to transform more than this number of items, you will need to implement your own logic to handle this.

##### Exceptions

[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) — thrown when the assigned value is less than 1.

**Introduced in:** 0.4.0


#### ReportingInterval

The number of milliseconds between progress reports. This property can be used to control how often progress is reported, which can be useful for performance tuning or to reduce the frequency of updates in a UI.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the number of milliseconds between progress reports.

##### Default Value

`1000` (1 second).

##### Exceptions

[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) — thrown when the assigned value is less than 1.

**Introduced in:** 0.4.0


#### SkipItemCount

The number of items to skip before returning items. This property can be used to ignore the first N items, which is useful for resuming partway through a pipeline or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the number of items to skip.

##### Default Value

`0` — no items are skipped. All items received from the upstream enumerable are transformed and yielded.

##### Exceptions

[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) — thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### Methods

| Method | Description |
|--------|-------------|
| `TransformAsync(IAsyncEnumerable<TSource> items)` | Runs the stage. *(virtual)* |
| `TransformAsync(IAsyncEnumerable<TSource> items, CancellationToken token)` | Runs the stage with cancellation. *(virtual)* |
| `TransformAsync(IAsyncEnumerable<TSource> items, IProgress<TProgress> progress)` | Runs the stage reporting progress. *(virtual)* |
| `TransformAsync(IAsyncEnumerable<TSource> items, IProgress<TProgress> progress, CancellationToken token)` | Runs the stage reporting progress, with cancellation. *(virtual)* |
| `TransformWorkerAsync(IAsyncEnumerable<TSource> items, CancellationToken token)` | *(protected abstract)* The worker that does the actual work; returns `IAsyncEnumerable<TDestination>`. |
| `CreateProgressReport()` | *(protected abstract)* Creates the `TProgress` report; called on the progress timer's thread. |
| `IncrementCurrentItemCount()` / `IncrementCurrentItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentItemCount`; call once per item transformed. |
| `IncrementCurrentSkippedItemCount()` / `IncrementCurrentSkippedItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentSkippedItemCount`. |
| `HandleItemError(ItemErrorContext context)` | *(protected)* Applies the error policy to a failed item (calls `OnItemError`, counts skipped failures) and returns the `ItemErrorAction`. |
| `OnItemError(ItemErrorContext context)` | *(protected virtual)* Decides what to do with a failed item; the default delegates to `ErrorPolicy`. |
| `WrapWorkerExecution(Func<CancellationToken, IAsyncEnumerable<TDestination>> workerFactory, CancellationToken token)` | *(protected virtual)* Resilience seam around every worker invocation; the default applies `WorkerResilience`. |
| `CreateProgressTimer(IProgress<TProgress> progress)` | *(protected virtual)* Creates the timer that drives progress callbacks. Tests no longer need to override it (see [TestKit](testkit.md#progress-timing-in-tests)). |
| `Dispose()` / `DisposeAsync()` / `Dispose(bool disposing)` | Releases resources. The base implementation is a no-op; override `Dispose(bool)` / `DisposeAsync()` when your stage owns resources. A disposed stage throws `ObjectDisposedException` when run. |


#### TransformAsync(IAsyncEnumerable\<TSource\> items)

Transforms data from the source format to the destination format by enumerating it asynchronously.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.


#### TransformAsync(IAsyncEnumerable\<TSource\> items, CancellationToken cancellationToken)

Transforms data from the source format to the destination format by enumerating it asynchronously with the ability to cancel.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.


#### TransformAsync(IAsyncEnumerable\<TSource\> items, IProgress\<TProgress\> progress)

Transforms data from the source format to the destination format by enumerating it asynchronously, periodically reporting progress.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the transformer.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.


#### TransformAsync(IAsyncEnumerable\<TSource\> items, IProgress\<TProgress\> progress, CancellationToken cancellationToken)

Transforms data asynchronously with both progress reporting and cancellation.

##### Parameters

`items` — [`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TSource`, representing the data to be transformed.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the transformer.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TDestination`, representing the transformed data.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0


## TransformerBase\<TSource, TDestination\>

A convenience subclass of `TransformerBase<TSource, TDestination, TProgress>` with `TProgress` fixed to the built-in `Report` type and a default `CreateProgressReport()`, so a derived class only implements `TransformWorkerAsync`. Use it when you do not need a custom progress type; override `CreateProgressReport()` to enrich the report (for example to set `TotalItemCount`). It has the same two constructors as the full base class.


## TransformerOptions

A `record` holding the construction-time configuration shared by every transformer: `MaximumItemCount` (default `int.MaxValue`, must be at least 1), `SkipItemCount` (default `0`, must be at least 0), `ReportingInterval` (default `1000` milliseconds, must be at least 1) and `ErrorPolicy`. All are `init`-only. Derive your own options record from it to add settings specific to your transformer.

```csharp
var options = new TransformerOptions
{
    SkipItemCount = 1,             // ignore the first item
    MaximumItemCount = 10_000,
    ReportingInterval = 500,
};
```
