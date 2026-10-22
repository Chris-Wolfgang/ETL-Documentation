# Loader

API reference for the loader interfaces and the `LoaderBase<TDestination, TProgress>` abstract base class in `Wolfgang.Etl.Abstractions`.

For a step-by-step guide to building a concrete loader, see [Your First Loader](../getting-started/your-first-loader.md). For the underlying architecture, see [Architecture](../architecture.md).

The loader is the final step in the ETL process, writing data to the final destination.


## Requirements

1. A loader should be implemented in a generic way, allowing it to be reused for different types of ETL processes. For example, a loader that writes data to a JSON file could be used to write any type of data to JSON regardless of the type.
   ```csharp
   // JsonLoader should be able to write Foo and Bar
   var loader1 = new JsonLoader<Foo>();
   var loader2 = new JsonLoader<Bar>();
   ```
1. Each `Loader<TDestination>` should be interchangeable for any other `Loader<TDestination>`, where `TDestination` is of the same type. This allows your application to swap loaders at compile time or runtime, making your ETL more flexible.
   ```csharp
   // jsonLoader and csvLoader should be interchangeable
   var jsonLoader = new JsonLoader<MyDataType>();
   var csvLoader = new CsvLoader<MyDataType>();
   ```
1. A loader is responsible for handling any exceptions that may occur during the loading process, including but not limited to network issues, permissions issues, space-full issues, and destination unavailability.
1. The loader should be resilient and capable of retrying operations in case of transient failures.
1. The data is passed to the loader as an asynchronous stream of type `TDestination`, allowing for efficient processing by the loader. Whenever possible, the loader should write 1 item at a time.

    > For performance reasons it may be necessary to buffer items and write them in small batches. However, the loader should not accumulate large amounts of data in memory before writing. For example, a loader that writes to a CSV file may find it more efficient to flush a few rows at a time, but should not hold the entire dataset in memory before writing, as some pipelines may contain gigabytes of data.
1. The loader should **not** do any transformation of the data. Transforming the data is the responsibility of the transformer. Its sole responsibility is to write data to the destination.

    > Occasionally, some minimal transformation may be necessary to ensure the data is in a suitable format for the destination. For example, a CSV loader might serialize each record to a delimited string before writing. This kind of format-specific serialization is acceptable; business-logic transformation belongs in a transformer.


## Interfaces

The four loader interfaces form a diamond — see [Architecture § Interfaces](../architecture.md#interfaces) for the shape shared across all three stages.

| Interface | Description |
|-----------|-------------|
| [ILoadAsync](#iloadasync) | A method to load data asynchronously to the destination. |
| [ILoadWithCancellationAsync](#iloadwithcancellationasync) | A method to load data asynchronously with the ability to cancel. |
| [ILoadWithProgressAsync](#iloadwithprogressasync) | A method to load data asynchronously while reporting progress. |
| [ILoadWithProgressAndCancellationAsync](#iloadwithprogressandcancellationasync) | A method to load data asynchronously with both progress reporting and cancellation. |


### ILoadAsync

#### Type Parameters

`TDestination` — the type of elements being passed in. This represents your destination or target data.

#### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items)` | Writes data to the destination |

#### LoadAsync\<TDestination\>

Asynchronously enumerates the items passed in and writes them to the destination.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be written.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

**Introduced in:** 0.4.0


### ILoadWithCancellationAsync

#### Type Parameters

`TDestination` — the type of elements being passed in. This represents your destination or target data.

#### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, CancellationToken cancellationToken)` | Writes data to the destination with the ability to cancel |

#### LoadAsync\<TDestination\> (CancellationToken token)

Asynchronously enumerates the items passed in and writes them to the destination.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be written.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

**Introduced in:** 0.4.0


### ILoadWithProgressAsync\<TDestination, TProgress\>

#### Type Parameters

`TDestination` — the type of elements being passed in. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress)` | Writes data to the destination reporting progress |

#### LoadAsync\<TDestination\>

Asynchronously enumerates the items passed in and writes them to the destination, periodically reporting progress.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be written.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the loader.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

**Introduced in:** 0.4.0


### ILoadWithProgressAndCancellationAsync\<TDestination, TProgress\>

#### Type Parameters

`TDestination` — the type of elements being passed in. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

#### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress, CancellationToken cancellationToken)` | Writes data to the destination reporting progress with the ability to cancel |

#### LoadAsync\<TDestination\>

Asynchronously enumerates the items passed in and writes them to the destination, periodically reporting progress, with the ability to cancel.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be written.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the loader.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

**Introduced in:** 0.4.0


## LoaderBase\<TDestination, TProgress\>

### Definition

Provides an abstract base class that implements [`ILoadAsync`](#iloadasync), [`ILoadWithCancellationAsync`](#iloadwithcancellationasync), [`ILoadWithProgressAsync`](#iloadwithprogressasync), and [`ILoadWithProgressAndCancellationAsync`](#iloadwithprogressandcancellationasync) with basic functionality, allowing for quickly building user-defined loaders. This abstract class only requires that you implement two methods: [`CreateProgressReport`](#createprogressreport) and [`LoadWorkerAsync`](#loadworkerasync).

### Type Parameters

`TDestination` — the type of elements being loaded. This represents your destination or target data.

`TProgress` — the type of object being used to report progress.

### Constructors

| Constructor | Description |
|-------------|-------------|
| `LoaderBase(LoaderOptions? options = null)` | *(protected)* Initializes the stage from an [`LoaderOptions`](#loaderoptions) record. `null` or omitted applies the defaults; out-of-range values throw `ArgumentOutOfRangeException` here. |
| `LoaderBase()` | *(protected, hidden from IntelliSense)* Kept for binary compatibility with assemblies compiled before the options constructor existed; same as passing `null`. |

### Properties

| Property | Description |
|----------|-------------|
| `CurrentItemCount` | *(get)* The number of items loaded so far in the current run. |
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
    `MaximumItemCount`, `SkipItemCount` and `ReportingInterval` still have public setters, but the setters are `[Obsolete]` and will be removed (ADR-0009): pass the values in the [`LoaderOptions`](#loaderoptions) record (or the library's own options record that derives from it, such as `JsonLineLoaderOptions`, `DbLoaderOptions`) when the stage is constructed. A value assigned after a run has started is not honoured.


#### CurrentItemCount

Read-only. The current number of items that have been loaded. This property is updated as items are loaded and can be used to report progress.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the current number of items that have been loaded.

##### Default Value

`0`.

**Introduced in:** 0.4.0


#### MaximumItemCount

The maximum number of items to be loaded before exiting and reporting complete. This property can be used to limit the number of items loaded, which is useful for testing or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the maximum number of items to load.

##### Default Value

[`int.MaxValue`](https://learn.microsoft.com/en-us/dotnet/api/system.int32.maxvalue) — by default there is no limit on the number of items loaded.

> If you are trying to load a large number of items, you are limited to the maximum value of a 32-bit integer, which is 2,147,483,647. If you need to load more than this number of items, you will need to implement your own logic to handle this.

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

The number of items to skip before writing items. This property can be used to ignore the first N items, which is useful for resuming partway through a pipeline or when only a subset of data is needed.

##### Property Value

[`int`](https://learn.microsoft.com/en-us/dotnet/api/system.int32) — the number of items to skip.

##### Default Value

`0` — no items are skipped. All items received from the pipeline are written to the destination.

##### Exceptions

[`ArgumentOutOfRangeException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception) — thrown when the assigned value is less than 0.

**Introduced in:** 0.4.0


### Methods

| Method | Description |
|--------|-------------|
| `LoadAsync(IAsyncEnumerable<TDestination> items)` | Runs the stage. *(virtual)* |
| `LoadAsync(IAsyncEnumerable<TDestination> items, CancellationToken token)` | Runs the stage with cancellation. *(virtual)* |
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress)` | Runs the stage reporting progress. *(virtual)* |
| `LoadAsync(IAsyncEnumerable<TDestination> items, IProgress<TProgress> progress, CancellationToken token)` | Runs the stage reporting progress, with cancellation. *(virtual)* |
| `LoadWorkerAsync(IAsyncEnumerable<TDestination> items, CancellationToken token)` | *(protected abstract)* The worker that does the actual work; returns `Task`. |
| `CreateProgressReport()` | *(protected abstract)* Creates the `TProgress` report; called on the progress timer's thread. |
| `IncrementCurrentItemCount()` / `IncrementCurrentItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentItemCount`; call once per item loaded. |
| `IncrementCurrentSkippedItemCount()` / `IncrementCurrentSkippedItemCount(int count)` | *(protected)* Thread-safe increment of `CurrentSkippedItemCount`. |
| `HandleItemError(ItemErrorContext context)` | *(protected)* Applies the error policy to a failed item (calls `OnItemError`, counts skipped failures) and returns the `ItemErrorAction`. |
| `OnItemError(ItemErrorContext context)` | *(protected virtual)* Decides what to do with a failed item; the default delegates to `ErrorPolicy`. |
| `WrapWorkerExecution(Func<CancellationToken, Task> workerFactory, CancellationToken token)` | *(protected virtual)* Resilience seam around every worker invocation; the default applies `WorkerResilience`. |
| `CreateProgressTimer(IProgress<TProgress> progress)` | *(protected virtual)* Creates the timer that drives progress callbacks. Tests no longer need to override it (see [TestKit](testkit.md#progress-timing-in-tests)). |
| `Dispose()` / `DisposeAsync()` / `Dispose(bool disposing)` | Releases resources. The base implementation is a no-op; override `Dispose(bool)` / `DisposeAsync()` when your stage owns resources. A disposed stage throws `ObjectDisposedException` when run. |


#### LoadAsync(IAsyncEnumerable\<TDestination\> items)

Writes data to the destination by asynchronously enumerating `items`.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be loaded.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.


#### LoadAsync(IAsyncEnumerable\<TDestination\> items, CancellationToken cancellationToken)

Writes data to the destination asynchronously with the ability to cancel.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be loaded.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.


#### LoadAsync(IAsyncEnumerable\<TDestination\> items, IProgress\<TProgress\> progress)

Writes data to the destination asynchronously, periodically reporting progress.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be loaded.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the loader.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.


#### LoadAsync(IAsyncEnumerable\<TDestination\> items, IProgress\<TProgress\> progress, CancellationToken cancellationToken)

Writes data to the destination asynchronously with both progress reporting and cancellation.

##### Parameters

`items` — [`IAsyncEnumerable<TDestination>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1), an asynchronous stream of items of type `TDestination`, representing the data to be loaded.

`progress` — [IProgress\<TProgress\>](https://learn.microsoft.com/en-us/dotnet/api/system.progress-1), an object that will be used to report the current progress of the loader.

`cancellationToken` — [CancellationToken](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken), the token to monitor for cancellation requests.

##### Returns

[`Task`](https://learn.microsoft.com/en-us/dotnet/fundamentals/runtime-libraries/system-threading-tasks-task) — a task that represents the asynchronous operation. The task will complete when all items have been written to the destination.

##### Exceptions

[`ArgumentNullException`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception) — thrown when the `progress` parameter is null.

**Introduced in:** 0.4.0


## LoaderBase\<TDestination\>

A convenience subclass of `LoaderBase<TDestination, TProgress>` with `TProgress` fixed to the built-in `Report` type and a default `CreateProgressReport()`, so a derived class only implements `LoadWorkerAsync`. Use it when you do not need a custom progress type; override `CreateProgressReport()` to enrich the report (for example to set `TotalItemCount`). It has the same two constructors as the full base class.


## LoaderOptions

A `record` holding the construction-time configuration shared by every loader: `MaximumItemCount` (default `int.MaxValue`, must be at least 1), `SkipItemCount` (default `0`, must be at least 0), `ReportingInterval` (default `1000` milliseconds, must be at least 1) and `ErrorPolicy`. All are `init`-only. Libraries derive their own options records from it, so the same properties appear on, for example, `JsonLineLoaderOptions`, `DbLoaderOptions`.

```csharp
var options = new LoaderOptions
{
    SkipItemCount = 1,             // ignore the first item
    MaximumItemCount = 10_000,
    ReportingInterval = 500,
};
```
