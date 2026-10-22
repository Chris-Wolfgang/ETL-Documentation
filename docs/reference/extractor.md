# Extractor

API reference for the extractor interfaces and the `ExtractorBase<TSource, TProgress>` abstract base class in `Wolfgang.Etl.Abstractions`.

For a step-by-step guide to building a concrete extractor, see [Your First Extractor](../getting-started/your-first-extractor.md). For the underlying architecture, see [Architecture](../architecture.md).

The extractor is the first step in the ETL process, responsible for pulling data from a source such as a database, CSV or JSON file, API, or web service.


## Requirements

1. An extractor should be implemented in a generic way, allowing it to be reused for different types of ETL processes. For example, an extractor for a CSV file could be used to extract data from any CSV file, regardless of its content.
   ```csharp
   // JsonExtractor should be able to retrieve Foo and Bar
   var extractor1 = new JsonExtractor<Foo>();
   var extractor2 = new JsonExtractor<Bar>();
   ```
1. Each `Extractor<T>` should be interchangeable for any other `Extractor<T>`. This allows your application to swap extractors at compile time or runtime, making your ETL more flexible.
   ```csharp
   // jsonExtractor and csvExtractor should be interchangeable
   var jsonExtractor = new JsonExtractor<MyDataType>();
   var csvExtractor = new CsvExtractor<MyDataType>();
   ```
1. An extractor is responsible for handling any exceptions that may occur during the extraction process, including but not limited to network issues, data format errors such as incomplete files, or source unavailability.
1. The extractor should be resilient and capable of retrying operations in case of transient failures.
1. The extracted data is returned as an asynchronous stream of type `TSource`, allowing for efficient processing by the transformer and loader components of the ETL pipeline. Whenever possible, the extractor should retrieve 1 item at a time.

    > For performance reasons it may be necessary to extract more than one item at a time and buffer them until they are requested. However, the extractor should not retrieve large amounts of data, which will consume large amounts of memory and other resources and delay processing of the first item. For example, an extractor that loads a CSV file may find it more efficient to prefetch a few rows at a time, but should not load the entire file at once, as some files may be gigabytes in size.
1. The extractor should **not** do any transformation of the data. Transforming the data is the responsibility of the transformer. Its sole responsibility is to retrieve data from the source and provide it in its raw form to the transformer.

    > Occasionally, some minimal transformation may be necessary to ensure the data is in a suitable format for further processing. For example, a JSON extractor might read a file of JSON and deserialize it into a list of objects, which would then be passed to the transformer for further processing. An alternative approach would be to pass the JSON as a string to a JSON transformer, which would then deserialize it into objects and pass it to the next transformer in the pipeline.


## Interfaces

The four extractor interfaces form a diamond — see [Architecture § Interfaces](../architecture.md#interfaces) for the shape shared across all three stages.

| Interface | Description |
|-----------|-------------|
| [IExtractAsync](#iextractasync) | A method to extract data asynchronously from a source. |
| [IExtractWithCancellationAsync](#iextractwithcancellationasync) | A method to extract data asynchronously with the ability to cancel. |
| [IExtractWithProgressAsync](#iextractwithprogressasync) | A method to extract data asynchronously while reporting progress. |
| [IExtractWithProgressAndCancellationAsync](#iextractwithprogressandcancellationasync) | A method to extract data asynchronously with both progress reporting and cancellation. |


### IExtractAsync

#### Type Parameters

`TSource` — the type of elements being returned. This represents your source data.

#### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync()` | Extracts data from the source asynchronously |

#### ExtractAsync\<TSource\>

Extracts data from the source by enumerating it asynchronously.

##### Parameters

None.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

**Introduced in:** 0.4.0


### IExtractWithCancellationAsync

#### Type Parameters

`TSource` — the type of elements being returned. This represents your source data.

#### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(CancellationToken cancellationToken)` | Extracts data asynchronously with the ability to cancel |

#### ExtractAsync\<TSource\>

Extracts data from the source by enumerating it asynchronously.

##### Parameters

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

**Introduced in:** 0.4.0


### IExtractWithProgressAsync\<TSource, TProgress\>

#### Type Parameters

`TSource` — the type of elements being returned. This represents your source data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(IProgress<TProgress> progress)` | Extracts data asynchronously reporting progress |

#### ExtractAsync\<TSource\>

Extracts data from the source by enumerating it asynchronously.

##### Parameters

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the extractor.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

**Introduced in:** 0.4.0


### IExtractWithProgressAndCancellationAsync\<TSource, TProgress\>

#### Type Parameters

`TSource` — the type of elements being returned. This represents your source data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync(IProgress<TProgress> progress, CancellationToken cancellationToken)` | Extracts data asynchronously reporting progress with the ability to cancel |

#### ExtractAsync\<TSource\>

Extracts data from the source by enumerating it asynchronously.

##### Parameters

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the extractor.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

**Introduced in:** 0.4.0


## ExtractorBase\<TSource, TProgress\>

### Definition

Provides an abstract base class that implements [`IExtractAsync`](#iextractasync), [`IExtractWithCancellationAsync`](#iextractwithcancellationasync), [`IExtractWithProgressAsync`](#iextractwithprogressasync), and [`IExtractWithProgressAndCancellationAsync`](#iextractwithprogressandcancellationasync) with basic functionality, allowing for quickly building user-defined extractors. This abstract class only requires that you implement two methods: [`CreateProgressReport`](#createprogressreport) and [`ExtractWorkerAsync`](#extractworkerasync).

### Type Parameters

`TSource` — the type of elements being returned. This represents your source data.

`TProgress` — the type of object being used to report progress.

### Constructors

| Constructor | Description |
|-------------|-------------|
| `ExtractorBase(ExtractorOptions? options = null)` | *(protected)* Initializes the stage from an [`ExtractorOptions`](#extractoroptions) record. `null` or omitted applies the defaults; out-of-range values throw `ArgumentOutOfRangeException` here. |
| `ExtractorBase()` | *(protected, hidden from IntelliSense)* Kept for binary compatibility with assemblies compiled before the options constructor existed; same as passing `null`. |

### Properties

| Property | Description |
|----------|-------------|
| `CurrentItemCount` | *(get)* The number of items extracted so far in the current run. |
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
    `MaximumItemCount`, `SkipItemCount` and `ReportingInterval` still have public setters, but the setters are `[Obsolete]` and will be removed (ADR-0009): pass the values in the [`ExtractorOptions`](#extractoroptions) record (or the library's own options record that derives from it, such as `JsonLineExtractorOptions`, `CsvExtractorOptions<T>`) when the stage is constructed. A value assigned after a run has started is not honoured.


#### CurrentItemCount

Read-only. The current number of items that have been extracted. This property is updated as items are extracted and can be used to report progress.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the current number of items that have been extracted.

##### Default Value

`0`.

**Introduced in:** 0.4.0


#### MaximumItemCount

The maximum number of items to be extracted before exiting and reporting complete. This property can be used to limit the number of items extracted, which is useful for testing or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the maximum number of items to extract.

##### Default Value

[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) — by default there is no limit on the number of items extracted.

> If you are trying to extract a large number of items, you are limited to the maximum value of a 32-bit integer, which is 2,147,483,647. If you need to extract more than this number of items, you will need to implement your own logic to handle this.

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

The number of items to skip before returning items. This property can be used to ignore the first N items, which is useful for resuming partway through a source or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the number of items to skip.

##### Default Value

`0` — no items are skipped. All items read from the source are returned.

##### Exceptions

[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) — thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### Methods

| Method | Description |
|--------|-------------|
| `ExtractAsync()` | Runs the stage. *(virtual)* |
| `ExtractAsync(CancellationToken token)` | Runs the stage with cancellation. *(virtual)* |
| `ExtractAsync(IProgress<TProgress> progress)` | Runs the stage reporting progress. *(virtual)* |
| `ExtractAsync(IProgress<TProgress> progress, CancellationToken token)` | Runs the stage reporting progress, with cancellation. *(virtual)* |
| `ExtractWorkerAsync(CancellationToken token)` | *(protected abstract)* The worker that does the actual work; returns `IAsyncEnumerable<TSource>`. |
| `CreateProgressReport()` | *(protected abstract)* Creates the `TProgress` report; called on the progress timer's thread. |
| `IncrementCurrentItemCount()` / `IncrementCurrentItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentItemCount`; call once per item extracted. |
| `IncrementCurrentSkippedItemCount()` / `IncrementCurrentSkippedItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentSkippedItemCount`. |
| `HandleItemError(ItemErrorContext context)` | *(protected)* Applies the error policy to a failed item (calls `OnItemError`, counts skipped failures) and returns the `ItemErrorAction`. |
| `OnItemError(ItemErrorContext context)` | *(protected virtual)* Decides what to do with a failed item; the default delegates to `ErrorPolicy`. |
| `WrapWorkerExecution(Func<CancellationToken, IAsyncEnumerable<TSource>> workerFactory, CancellationToken token)` | *(protected virtual)* Resilience seam around every worker invocation; the default applies `WorkerResilience`. |
| `CreateProgressTimer(IProgress<TProgress> progress)` | *(protected virtual)* Creates the timer that drives progress callbacks. Tests no longer need to override it (see [TestKit](testkit.md#progress-timing-in-tests)). |
| `Dispose()` / `DisposeAsync()` / `Dispose(bool disposing)` | Releases resources. The base implementation is a no-op; override `Dispose(bool)` / `DisposeAsync()` when your stage owns resources. A disposed stage throws `ObjectDisposedException` when run. |


#### ExtractAsync()

Extracts data from the source by enumerating it asynchronously.

##### Parameters

None.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.


#### ExtractAsync(CancellationToken cancellationToken)

Extracts data from the source by enumerating it asynchronously with the ability to cancel.

##### Parameters

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.


#### ExtractAsync(IProgress\<TProgress\> progress)

Extracts data from the source by enumerating it asynchronously, periodically reporting progress.

##### Parameters

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the extractor.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.


#### ExtractAsync(IProgress\<TProgress\> progress, CancellationToken cancellationToken)

Extracts data from the source asynchronously with both progress reporting and cancellation.

##### Parameters

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the extractor.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`IAsyncEnumerable<TSource>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) — an asynchronous stream of items of type `TSource`, representing the extracted data.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0


## ExtractorBase\<TSource\>

A convenience subclass of `ExtractorBase<TSource, TProgress>` with `TProgress` fixed to the built-in `Report` type and a default `CreateProgressReport()`, so a derived class only implements `ExtractWorkerAsync`. Use it when you do not need a custom progress type; override `CreateProgressReport()` to enrich the report (for example to set `TotalItemCount`). It has the same two constructors as the full base class.


## ExtractorOptions

A `record` holding the construction-time configuration shared by every extractor: `MaximumItemCount` (default `int.MaxValue`, must be at least 1), `SkipItemCount` (default `0`, must be at least 0), `ReportingInterval` (default `1000` milliseconds, must be at least 1) and `ErrorPolicy`. All are `init`-only. Libraries derive their own options records from it, so the same properties appear on, for example, `JsonLineExtractorOptions`, `CsvExtractorOptions<T>`.

```csharp
var options = new ExtractorOptions
{
    SkipItemCount = 1,             // ignore the first item
    MaximumItemCount = 10_000,
    ReportingInterval = 500,
};
```
